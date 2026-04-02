# Error Handling

## Description
Covers every mechanism COBOL provides for detecting, reporting, and recovering from errors at run time. This includes FILE STATUS codes for I/O operations, DECLARATIVES/USE AFTER procedures, imperative-scope error phrases (ON SIZE ERROR, ON OVERFLOW, ON EXCEPTION, AT END, INVALID KEY), the RETURN-CODE special register, and practical recovery patterns. Reference this file whenever a program must react to an abnormal condition rather than simply abending.

## Table of Contents
- [Core Concepts](#core-concepts)
  - [Error Handling Philosophy in COBOL](#error-handling-philosophy-in-cobol)
  - [FILE STATUS Codes](#file-status-codes)
  - [DECLARATIVES and USE AFTER Procedures](#declaratives-and-use-after-procedures)
  - [Imperative Error Phrases](#imperative-error-phrases)
  - [RETURN-CODE Special Register](#return-code-special-register)
- [Syntax & Examples](#syntax--examples)
  - [FILE STATUS Declaration and Checking](#file-status-declaration-and-checking)
  - [DECLARATIVES Section](#declaratives-section)
  - [ON SIZE ERROR / NOT ON SIZE ERROR](#on-size-error--not-on-size-error)
  - [ON OVERFLOW / NOT ON OVERFLOW](#on-overflow--not-on-overflow)
  - [ON EXCEPTION / NOT ON EXCEPTION](#on-exception--not-on-exception)
  - [AT END / NOT AT END](#at-end--not-at-end)
  - [INVALID KEY / NOT INVALID KEY](#invalid-key--not-invalid-key)
  - [RETURN-CODE Usage](#return-code-usage)
- [Common Patterns](#common-patterns)
- [Gotchas](#gotchas)
- [Related Topics](#related-topics)

## Core Concepts

### Error Handling Philosophy in COBOL

COBOL does not use exceptions, try/catch blocks, or signals. Instead, error handling is **inline and declarative**. Every I/O statement can carry conditional phrases (AT END, INVALID KEY), every arithmetic statement can carry ON SIZE ERROR, and every string statement can carry ON OVERFLOW or ON EXCEPTION. The programmer is responsible for testing conditions explicitly after each operation.

Two complementary strategies exist:

1. **Inline checking** -- test FILE STATUS or use conditional phrases immediately after the statement that might fail.
2. **Centralized checking** -- use DECLARATIVES/USE AFTER procedures to intercept errors in a dedicated section before control returns to the mainline.

When neither strategy is coded, the runtime typically abends on serious I/O errors or silently truncates on arithmetic overflow, both of which are dangerous in production.

### FILE STATUS Codes

FILE STATUS is a two-byte alphanumeric field declared in the FILE-CONTROL paragraph via the FILE STATUS IS clause. After every I/O operation on that file, the runtime places a two-character code in the field. The first character (status key 1) gives the category; the second character (status key 2) gives the detail.

#### Status Key 1 Categories

| Key 1 | Category |
|-------|----------|
| 0 | Successful completion |
| 1 | AT END condition |
| 2 | INVALID KEY condition |
| 3 | Permanent I/O error |
| 4 | Logic error (program bug) |
| 9 | Implementor-defined |

See [file_handling.md](file_handling.md) for the complete file status code reference table.

### DECLARATIVES and USE AFTER Procedures

DECLARATIVES is a special section at the beginning of the PROCEDURE DIVISION that contains one or more USE AFTER procedures. These procedures are invoked automatically by the runtime when an I/O error occurs, **before** control returns to the statement following the failed I/O verb.

Key characteristics:
- DECLARATIVES must appear **first** in the PROCEDURE DIVISION, before any other section.
- Each USE AFTER procedure is a separate section with its own USE sentence.
- A USE AFTER procedure can reference a specific file name or an entire file category (INPUT, OUTPUT, I-O, EXTEND).
- The body of the procedure executes automatically on error; the programmer does not call it explicitly.
- After the DECLARATIVES procedure completes, control returns to the statement following the I/O verb that triggered it (unless STOP RUN or GO TO is used within the DECLARATIVES section to divert control).
- DECLARATIVES procedures cannot execute I/O statements on the file that triggered them.

### Imperative Error Phrases

COBOL attaches optional error/success phrases to specific verb categories:

| Phrase | Applies To | Triggered When |
|--------|-----------|----------------|
| ON SIZE ERROR | ADD, SUBTRACT, MULTIPLY, DIVIDE, COMPUTE | Result exceeds receiving field size, or division by zero |
| NOT ON SIZE ERROR | ADD, SUBTRACT, MULTIPLY, DIVIDE, COMPUTE | Operation completes without overflow or division by zero |
| ON OVERFLOW | STRING, UNSTRING, CALL | Receiving field too small (STRING/UNSTRING) or called program not found (CALL) |
| NOT ON OVERFLOW | STRING, UNSTRING | Operation completes without overflow |
| ON EXCEPTION | CALL, ACCEPT, DISPLAY | Called program not found (CALL) or platform-specific error (ACCEPT/DISPLAY) |
| NOT ON EXCEPTION | CALL, ACCEPT, DISPLAY | Operation succeeds |
| AT END | READ, RETURN, SEARCH | End of file (READ/RETURN) or end of table (SEARCH) |
| NOT AT END | READ, RETURN, SEARCH | Record successfully retrieved / entry found |
| INVALID KEY | READ, WRITE, REWRITE, DELETE, START | Key condition error on indexed/relative file |
| NOT INVALID KEY | READ, WRITE, REWRITE, DELETE, START | Keyed operation succeeds |

These phrases create a scope: the statements between ON SIZE ERROR and END-verb (or NOT ON SIZE ERROR) form an **imperative scope** that executes only when the condition is met. This is the primary inline error handling mechanism.

### RETURN-CODE Special Register

RETURN-CODE is a compiler-maintained binary fullword (typically PIC S9(4) COMP, though some implementations use S9(8) COMP) that serves as the inter-program return code. It is:

- Set by a called program before it returns (via GOBACK or STOP RUN) to communicate success or failure to the caller.
- Inspected by the calling program after a CALL to determine the outcome.
- Passed back to the operating system when the main program ends, becoming the job step return code (condition code) in JCL.
- Conventional values: 0 = success, 4 = warning, 8 = error, 12 = severe error, 16 = terminal error.

RETURN-CODE is **not** automatically set by COBOL I/O statements. It is solely for program-to-program and program-to-OS communication.

## Syntax & Examples

### FILE STATUS Declaration and Checking

```cobol
       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT CUSTOMER-FILE
               ASSIGN TO 'CUSTFILE'
               ORGANIZATION IS INDEXED
               ACCESS MODE IS DYNAMIC
               RECORD KEY IS CUST-ID
               FILE STATUS IS WS-CUST-STATUS.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-CUST-STATUS        PIC XX.
           88 CUST-SUCCESS        VALUE '00'.
           88 CUST-DUPLICATE      VALUE '22'.
           88 CUST-NOT-FOUND      VALUE '23'.
           88 CUST-EOF            VALUE '10'.

       PROCEDURE DIVISION.
           OPEN I-O CUSTOMER-FILE
           IF NOT CUST-SUCCESS
               DISPLAY 'OPEN ERROR: ' WS-CUST-STATUS
               STOP RUN
           END-IF

           READ CUSTOMER-FILE
               KEY IS CUST-ID
           END-READ
           EVALUATE TRUE
               WHEN CUST-SUCCESS
                   PERFORM PROCESS-CUSTOMER
               WHEN CUST-NOT-FOUND
                   DISPLAY 'CUSTOMER NOT FOUND'
               WHEN OTHER
                   DISPLAY 'READ ERROR: ' WS-CUST-STATUS
                   PERFORM ERROR-ABORT
           END-EVALUATE
```

Using 88-level condition names on the FILE STATUS field is a widely recommended practice. It makes the code self-documenting and avoids magic literals scattered throughout the program.

### DECLARATIVES Section

```cobol
       PROCEDURE DIVISION.
       DECLARATIVES.

       CUST-FILE-ERROR SECTION.
           USE AFTER STANDARD ERROR PROCEDURE ON CUSTOMER-FILE.
       CUST-FILE-ERROR-PARA.
           DISPLAY 'I/O ERROR ON CUSTOMER-FILE'
           DISPLAY 'FILE STATUS: ' WS-CUST-STATUS
           DISPLAY 'OPERATION ATTEMPTED AT: ' WS-CURRENT-PARA
           SET WS-IO-ERROR-FLAG TO TRUE
           .

       INPUT-FILE-ERROR SECTION.
           USE AFTER STANDARD ERROR PROCEDURE ON INPUT.
       INPUT-FILE-ERROR-PARA.
           DISPLAY 'I/O ERROR ON AN INPUT FILE'
           DISPLAY 'FILE STATUS: ' WS-FILE-STATUS
           SET WS-IO-ERROR-FLAG TO TRUE
           .

       END DECLARATIVES.

       MAIN-LOGIC SECTION.
       MAIN-PARA.
           OPEN INPUT TRANSACTION-FILE
           OPEN I-O CUSTOMER-FILE
           PERFORM PROCESS-TRANSACTIONS
               UNTIL WS-EOF OR WS-IO-ERROR-FLAG
           CLOSE TRANSACTION-FILE
           CLOSE CUSTOMER-FILE
           STOP RUN.
```

The USE AFTER clause supports these variants:
- `USE AFTER STANDARD ERROR PROCEDURE ON file-name` -- specific file
- `USE AFTER STANDARD ERROR PROCEDURE ON INPUT` -- all files opened for input
- `USE AFTER STANDARD ERROR PROCEDURE ON OUTPUT` -- all files opened for output
- `USE AFTER STANDARD ERROR PROCEDURE ON I-O` -- all files opened for I-O
- `USE AFTER STANDARD ERROR PROCEDURE ON EXTEND` -- all files opened for extend
- `USE AFTER STANDARD EXCEPTION PROCEDURE ON ...` -- synonymous with ERROR

A file-specific USE takes precedence over a category-wide USE when both are present.

### ON SIZE ERROR / NOT ON SIZE ERROR

```cobol
       COMPUTE WS-AVERAGE = WS-TOTAL / WS-COUNT
           ON SIZE ERROR
               DISPLAY 'ARITHMETIC ERROR: OVERFLOW OR DIVIDE BY ZERO'
               MOVE ZEROS TO WS-AVERAGE
               SET WS-CALC-ERROR TO TRUE
           NOT ON SIZE ERROR
               SET WS-CALC-OK TO TRUE
       END-COMPUTE

       ADD WS-AMOUNT TO WS-RUNNING-TOTAL
           ON SIZE ERROR
               DISPLAY 'TOTAL OVERFLOW, AMOUNT: ' WS-AMOUNT
               PERFORM OVERFLOW-RECOVERY
       END-ADD

       DIVIDE WS-TOTAL BY WS-DIVISOR
           GIVING WS-RESULT REMAINDER WS-REMAIN
           ON SIZE ERROR
               DISPLAY 'DIVISION ERROR'
               MOVE ZEROS TO WS-RESULT
                                  WS-REMAIN
           NOT ON SIZE ERROR
               CONTINUE
       END-DIVIDE
```

ON SIZE ERROR triggers when:
- The result of an arithmetic operation is too large for the receiving field.
- A DIVIDE operation attempts division by zero.
- An intermediate result overflows during COMPUTE evaluation.

When ON SIZE ERROR fires, the receiving field is **unchanged** -- it retains its value from before the statement.

### ON OVERFLOW / NOT ON OVERFLOW

```cobol
       STRING WS-LAST-NAME DELIMITED BY SPACES
              ', '           DELIMITED BY SIZE
              WS-FIRST-NAME DELIMITED BY SPACES
           INTO WS-FULL-NAME
           WITH POINTER WS-STR-PTR
           ON OVERFLOW
               DISPLAY 'NAME TRUNCATED AT POSITION ' WS-STR-PTR
               SET WS-NAME-TRUNCATED TO TRUE
           NOT ON OVERFLOW
               SET WS-NAME-COMPLETE TO TRUE
       END-STRING

       UNSTRING WS-INPUT-LINE
           DELIMITED BY ',' OR SPACES
           INTO WS-FIELD-1 WS-FIELD-2 WS-FIELD-3
           ON OVERFLOW
               DISPLAY 'MORE FIELDS THAN EXPECTED IN INPUT'
       END-UNSTRING
```

For STRING, overflow means the pointer has exceeded the length of the receiving field and data was lost. For UNSTRING, overflow means there are more delimited fields in the source than receiving fields specified.

Note: For the CALL statement, ON OVERFLOW and ON EXCEPTION are functionally identical and both detect the condition where the called program cannot be found.

### ON EXCEPTION / NOT ON EXCEPTION

```cobol
       CALL WS-PROGRAM-NAME
           USING WS-INPUT-RECORD
                 WS-OUTPUT-RECORD
           ON EXCEPTION
               DISPLAY 'PROGRAM NOT FOUND: ' WS-PROGRAM-NAME
               SET WS-CALL-FAILED TO TRUE
           NOT ON EXCEPTION
               SET WS-CALL-OK TO TRUE
               EVALUATE RETURN-CODE
                   WHEN 0
                       CONTINUE
                   WHEN 4
                       DISPLAY 'WARNING FROM ' WS-PROGRAM-NAME
                   WHEN 8
                       DISPLAY 'ERROR FROM ' WS-PROGRAM-NAME
                       SET WS-SUB-ERROR TO TRUE
                   WHEN OTHER
                       DISPLAY 'SEVERE ERROR FROM ' WS-PROGRAM-NAME
                       PERFORM ERROR-ABORT
               END-EVALUATE
       END-CALL
```

ON EXCEPTION on a CALL fires when the runtime cannot load or find the target program. It does **not** fire when the called program abends or returns a non-zero RETURN-CODE. You must check RETURN-CODE yourself after a successful CALL.

### AT END / NOT AT END

```cobol
       PERFORM UNTIL WS-EOF
           READ TRANSACTION-FILE INTO WS-TRANS-RECORD
               AT END
                   SET WS-EOF TO TRUE
               NOT AT END
                   ADD 1 TO WS-RECORD-COUNT
                   PERFORM PROCESS-TRANSACTION
           END-READ
       END-PERFORM

       SEARCH WS-STATE-TABLE
           AT END
               DISPLAY 'STATE CODE NOT FOUND: ' WS-SEARCH-CODE
               MOVE 'UNKNOWN' TO WS-STATE-NAME
           WHEN WS-STATE-CODE(WS-IDX) = WS-SEARCH-CODE
               MOVE WS-STATE-NAME(WS-IDX) TO WS-RESULT-NAME
       END-SEARCH
```

For READ, AT END corresponds to FILE STATUS '10'. For SEARCH (serial), AT END fires when the entire table has been scanned without a match. For SEARCH ALL (binary), AT END fires when no matching entry is found.

### INVALID KEY / NOT INVALID KEY

```cobol
       WRITE CUST-RECORD
           INVALID KEY
               IF CUST-DUPLICATE
                   DISPLAY 'DUPLICATE KEY: ' CUST-ID
                   PERFORM HANDLE-DUPLICATE
               ELSE
                   DISPLAY 'WRITE ERROR: ' WS-CUST-STATUS
                   PERFORM ERROR-ABORT
               END-IF
           NOT INVALID KEY
               ADD 1 TO WS-RECORDS-WRITTEN
       END-WRITE

       DELETE CUSTOMER-FILE RECORD
           INVALID KEY
               DISPLAY 'RECORD NOT FOUND FOR DELETE: ' CUST-ID
           NOT INVALID KEY
               ADD 1 TO WS-RECORDS-DELETED
       END-DELETE

       START CUSTOMER-FILE
           KEY IS GREATER THAN CUST-ID
           INVALID KEY
               DISPLAY 'NO RECORD WITH KEY > ' CUST-ID
               SET WS-NO-MORE TO TRUE
       END-START
```

INVALID KEY fires for any FILE STATUS code in the 2x range (21, 22, 23, 24). The FILE STATUS field is populated before the INVALID KEY phrase executes, so you can inspect it inside the phrase to determine the exact cause.

### RETURN-CODE Usage

```cobol
      *---------------------------------------------------------
      * CALLING PROGRAM
      *---------------------------------------------------------
       CALL 'VALIDATE' USING WS-INPUT-DATA
       END-CALL
       EVALUATE RETURN-CODE
           WHEN 0
               PERFORM CONTINUE-PROCESSING
           WHEN 4
               DISPLAY 'VALIDATION WARNINGS FOUND'
               PERFORM CONTINUE-PROCESSING
           WHEN 8
               DISPLAY 'VALIDATION ERRORS FOUND'
               PERFORM WRITE-ERROR-REPORT
           WHEN 16
               DISPLAY 'VALIDATION CATASTROPHIC FAILURE'
               PERFORM ERROR-ABORT
       END-EVALUATE

      *---------------------------------------------------------
      * CALLED PROGRAM (VALIDATE)
      *---------------------------------------------------------
       IDENTIFICATION DIVISION.
       PROGRAM-ID. VALIDATE.
       DATA DIVISION.
       LINKAGE SECTION.
       01  LS-INPUT-DATA         PIC X(100).

       PROCEDURE DIVISION USING LS-INPUT-DATA.
           IF LS-INPUT-DATA = SPACES
               MOVE 8 TO RETURN-CODE
               GOBACK
           END-IF
           PERFORM VALIDATION-LOGIC
           IF WS-WARNINGS-FOUND
               MOVE 4 TO RETURN-CODE
           ELSE
               MOVE 0 TO RETURN-CODE
           END-IF
           GOBACK.
```

## Common Patterns

### Centralized I/O Error Routine

A single paragraph that handles all file errors, called after every I/O statement:

```cobol
       01  WS-IO-STATUS          PIC XX.
       01  WS-IO-OPERATION       PIC X(10).
       01  WS-IO-FILE-NAME       PIC X(30).

       CHECK-IO-STATUS.
           IF WS-IO-STATUS NOT = '00' AND '10'
               DISPLAY '*** I/O ERROR ***'
               DISPLAY 'FILE:      ' WS-IO-FILE-NAME
               DISPLAY 'OPERATION: ' WS-IO-OPERATION
               DISPLAY 'STATUS:    ' WS-IO-STATUS
               PERFORM ERROR-ABORT
           END-IF.

       READ-NEXT-TRANSACTION.
           MOVE 'READ'        TO WS-IO-OPERATION
           MOVE 'TRANS-FILE'  TO WS-IO-FILE-NAME
           READ TRANSACTION-FILE INTO WS-TRANS-REC
           MOVE WS-TRANS-STATUS TO WS-IO-STATUS
           PERFORM CHECK-IO-STATUS.
```

### Retry Pattern for Locked Resources

Common in CICS and multi-user environments:

```cobol
       01  WS-RETRY-COUNT        PIC 9(2) VALUE 0.
       01  WS-MAX-RETRIES        PIC 9(2) VALUE 5.

       READ-WITH-RETRY.
           MOVE 0 TO WS-RETRY-COUNT
           PERFORM ATTEMPT-READ
               UNTIL CUST-SUCCESS
                  OR WS-RETRY-COUNT >= WS-MAX-RETRIES
           IF NOT CUST-SUCCESS
               DISPLAY 'FAILED AFTER ' WS-MAX-RETRIES ' RETRIES'
               PERFORM ERROR-ABORT
           END-IF.

       ATTEMPT-READ.
           READ CUSTOMER-FILE INTO WS-CUST-RECORD
               KEY IS CUST-ID
           END-READ
           IF WS-CUST-STATUS = '93'
               ADD 1 TO WS-RETRY-COUNT
           END-IF.
```

### Graceful File-Not-Found Handling

```cobol
       SELECT OPTIONAL PARAM-FILE
           ASSIGN TO 'PARMFILE'
           ORGANIZATION IS LINE SEQUENTIAL
           FILE STATUS IS WS-PARM-STATUS.

       01  WS-PARM-STATUS        PIC XX.
           88 PARM-OK            VALUE '00'.
           88 PARM-NOT-PRESENT   VALUE '05'.
           88 PARM-NOT-FOUND     VALUE '35'.

       LOAD-PARAMETERS.
           OPEN INPUT PARAM-FILE
           EVALUATE TRUE
               WHEN PARM-OK
                   PERFORM READ-PARAMETERS
               WHEN PARM-NOT-PRESENT
                   DISPLAY 'PARAMETER FILE NOT FOUND, USING DEFAULTS'
                   PERFORM SET-DEFAULT-PARAMETERS
               WHEN PARM-NOT-FOUND
                   DISPLAY 'PARAMETER FILE NOT FOUND, USING DEFAULTS'
                   PERFORM SET-DEFAULT-PARAMETERS
               WHEN OTHER
                   DISPLAY 'PARAM FILE OPEN ERROR: ' WS-PARM-STATUS
                   PERFORM ERROR-ABORT
           END-EVALUATE.
```

The SELECT OPTIONAL clause tells the runtime the file is not required to exist. Without it, a missing file causes a status '35' or an abend, depending on the runtime.

### Return Code Accumulator Pattern

Programs that call multiple subprograms often track the highest return code:

```cobol
       01  WS-MAX-RC             PIC S9(4) COMP VALUE 0.
       01  WS-STEP-RC            PIC S9(4) COMP VALUE 0.

       CALL 'STEP1' USING WS-DATA
       MOVE RETURN-CODE TO WS-STEP-RC
       IF WS-STEP-RC > WS-MAX-RC
           MOVE WS-STEP-RC TO WS-MAX-RC
       END-IF

       IF WS-MAX-RC < 8
           CALL 'STEP2' USING WS-DATA
           MOVE RETURN-CODE TO WS-STEP-RC
           IF WS-STEP-RC > WS-MAX-RC
               MOVE WS-STEP-RC TO WS-MAX-RC
           END-IF
       END-IF

       MOVE WS-MAX-RC TO RETURN-CODE
       STOP RUN.
```

### Defensive CLOSE Pattern

```cobol
       PROGRAM-CLEANUP.
           IF WS-INFILE-OPEN
               CLOSE INPUT-FILE
               SET WS-INFILE-OPEN TO FALSE
           END-IF
           IF WS-OUTFILE-OPEN
               CLOSE OUTPUT-FILE
               SET WS-OUTFILE-OPEN TO FALSE
           END-IF
           IF WS-ERRFILE-OPEN
               CLOSE ERROR-FILE
               SET WS-ERRFILE-OPEN TO FALSE
           END-IF.
```

Tracking open/closed state with flags avoids FILE STATUS '42' (closing a file that is not open) during error recovery paths where the program exits early.

### Error Logging to a Dedicated Error File

```cobol
       01  WS-ERROR-RECORD.
           05 WS-ERR-TIMESTAMP   PIC X(26).
           05 WS-ERR-PROGRAM     PIC X(8).
           05 WS-ERR-PARAGRAPH   PIC X(30).
           05 WS-ERR-FILE-NAME   PIC X(30).
           05 WS-ERR-STATUS      PIC XX.
           05 WS-ERR-MESSAGE     PIC X(80).

       LOG-ERROR.
           MOVE FUNCTION CURRENT-DATE TO WS-ERR-TIMESTAMP
           MOVE 'CUSTUPDT' TO WS-ERR-PROGRAM
           WRITE ERROR-RECORD FROM WS-ERROR-RECORD
           .
```

## Gotchas

- **Unchecked FILE STATUS is the number-one cause of silent data corruption.** If you do not check FILE STATUS after every I/O operation, a failed WRITE or REWRITE can go unnoticed while the program continues as if the record was stored successfully.

- **ON SIZE ERROR does not fire without explicit scope.** If you write `ADD A TO B ON SIZE ERROR DISPLAY 'ERR'` without the `END-ADD` scope terminator, the behavior depends on the coding style (period-delimited vs. explicit scope). Always use explicit scope terminators (END-ADD, END-COMPUTE, etc.) to guarantee the error phrase is correctly associated.

- **DECLARATIVES and inline phrases are independent.** If both a DECLARATIVES USE AFTER procedure and an INVALID KEY phrase exist for the same file, the DECLARATIVES procedure fires first, then the INVALID KEY phrase executes. They do not suppress each other. However, if an AT END or INVALID KEY phrase is present, the DECLARATIVES procedure does **not** fire for that specific condition -- only for conditions not covered by the inline phrase.

- **RETURN-CODE is global and fragile.** Any CALL (including runtime library calls and intrinsic functions on some platforms) may overwrite RETURN-CODE. Always MOVE RETURN-CODE to a working-storage field immediately after each CALL before performing any other operation.

- **FILE STATUS '23' on OPEN is a surprise.** When opening an OPTIONAL file for I-O and the file does not exist, some runtimes return '23' instead of '05'. Defensive code should check for both.

- **Status '00' after WRITE does not mean the data is on disk.** Most runtimes buffer writes. A '00' on WRITE means the data was accepted into the buffer. A failure can still occur on CLOSE when buffers are flushed. Always check FILE STATUS after CLOSE.

- **Division by zero behavior without ON SIZE ERROR is implementation-defined.** Some runtimes abend with a S0C7/S0CB, others produce unpredictable results in the receiving field. Never divide without either ON SIZE ERROR or a pre-check that the divisor is non-zero.

- **ON OVERFLOW on CALL is deprecated in favor of ON EXCEPTION.** Both are functionally equivalent for CALL, but ON EXCEPTION is the modern standard form. Using ON OVERFLOW on CALL still works but may confuse readers who associate ON OVERFLOW with STRING/UNSTRING.

- **AT END and INVALID KEY are mutually exclusive by context.** AT END applies to sequential reads; INVALID KEY applies to keyed operations. In DYNAMIC access mode with an indexed file, a sequential READ (READ file NEXT) uses AT END, while a random READ (READ file KEY IS) uses INVALID KEY. Confusing the two will cause compilation errors.

- **DECLARATIVES sections cannot perform paragraphs outside DECLARATIVES, and non-DECLARATIVES code cannot perform paragraphs inside DECLARATIVES.** This isolation is enforced by the compiler and can surprise programmers who try to share utility paragraphs.

- **FILE STATUS is set even when DECLARATIVES or inline phrases handle the error.** The status code is always populated regardless of whether error handling code exists. This means you can safely use FILE STATUS as a secondary check even when AT END or INVALID KEY is coded.

- **NOT ON SIZE ERROR / NOT AT END / etc. execute only when the operation succeeds.** They are not simply the "else" branch -- they specifically mean "the operation completed without the error condition." If a different, uncovered error occurs, neither branch may execute.

## Related Topics

- **file_handling.md** -- FILE STATUS codes are declared as part of the file definition; file_handling covers SELECT, OPEN, CLOSE, READ, WRITE, REWRITE, and DELETE operations that produce these status codes.
- **arithmetic.md** -- ON SIZE ERROR applies to all arithmetic verbs (ADD, SUBTRACT, MULTIPLY, DIVIDE, COMPUTE) covered in that file.
- **string_handling.md** -- ON OVERFLOW applies to STRING and UNSTRING operations documented there.
- **debugging.md** -- Error handling and debugging are complementary; debugging covers DISPLAY for diagnostics, USE FOR DEBUGGING declaratives, and trace techniques.
- **vsam.md** -- VSAM files produce the 9x series of implementor-defined FILE STATUS codes; vsam.md covers VSAM-specific error handling and the extended file status (VSAM return code, function code, feedback code).
