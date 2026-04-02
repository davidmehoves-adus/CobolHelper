# Debugging

## Description
Covers techniques and tools for diagnosing COBOL program failures at compile time and run time. Reference this file when a program abends, produces incorrect output, encounters data exceptions, or when you need to add diagnostic instrumentation to COBOL source code.

## Table of Contents
- [Core Concepts](#core-concepts)
  - [Debugging Philosophy in COBOL](#debugging-philosophy-in-cobol)
  - [System Abend Codes vs. User Abend Codes](#system-abend-codes-vs-user-abend-codes)
  - [Compile-Time vs. Run-Time Errors](#compile-time-vs-run-time-errors)
- [Syntax & Examples](#syntax--examples)
  - [DISPLAY Statement Debugging](#display-statement-debugging)
  - [EXHIBIT Statement](#exhibit-statement)
  - [READY TRACE and RESET TRACE](#ready-trace-and-reset-trace)
  - [USE FOR DEBUGGING Declarative](#use-for-debugging-declarative)
  - [Debugging Declaratives](#debugging-declaratives)
  - [File Status Checking Patterns](#file-status-checking-patterns)
- [Common Patterns](#common-patterns)
  - [Defensive Coding Patterns](#defensive-coding-patterns)
  - [Abend Code Reference Table](#abend-code-reference-table)
  - [Reading Dumps and SYSUDUMP](#reading-dumps-and-sysudump)
  - [Compiler Diagnostics](#compiler-diagnostics)
  - [Common Runtime Errors](#common-runtime-errors)
- [Gotchas](#gotchas)
- [Related Topics](#related-topics)

## Core Concepts

### Debugging Philosophy in COBOL

COBOL debugging differs fundamentally from debugging in languages with integrated development environments. Mainframe COBOL programs typically run in batch or CICS environments where interactive debugging may be limited or unavailable. Debugging therefore relies on a combination of:

1. **Preventive coding** -- defensive patterns that catch errors before they cascade.
2. **Diagnostic instrumentation** -- DISPLAY, EXHIBIT, and TRACE statements embedded in source code.
3. **Post-mortem analysis** -- reading abend codes, dumps, and compiler listings after a failure.
4. **Declarative debugging** -- the COBOL-standard USE FOR DEBUGGING facility.

The most effective COBOL debugging strategy is layered: write defensively, instrument selectively, and know how to read the evidence when something still goes wrong.

### System Abend Codes vs. User Abend Codes

Abend (abnormal end) codes fall into two categories:

- **System abend codes** begin with "S" and are issued by the operating system or subsystem. They use a three-character hexadecimal code (e.g., S0C7, S0C4). The leading "S" stands for "system."
- **User abend codes** begin with "U" and are issued by application programs via the ABEND macro or COBOL STOP RUN with a non-zero return code. They use a four-digit decimal code (e.g., U4038, U1026).

When diagnosing a failure, the first step is always to identify whether the abend is system or user, then look up the specific code.

### Compile-Time vs. Run-Time Errors

**Compile-time errors** are caught by the COBOL compiler and appear in the compiler listing. They are classified by severity:

| Severity | Code | Meaning |
|----------|------|---------|
| Informational | I | Condition noted; no action required |
| Warning | W | Possible error; program will compile |
| Error | E | Definite error; program may compile but results are unpredictable |
| Severe | S | Serious error; statement discarded or replaced |
| Unrecoverable | U | Fatal error; compilation terminates |

**Run-time errors** occur during program execution. They produce abend codes and, depending on configuration, storage dumps. Run-time errors are harder to diagnose because the program has already been compiled and the error depends on actual data values and system state.

## Syntax & Examples

### DISPLAY Statement Debugging

The DISPLAY statement is the most universally available debugging tool in COBOL. It writes data to SYSOUT (batch) or the terminal (interactive).

**Basic variable display:**

```cobol
       PROCEDURE DIVISION.
           DISPLAY 'STARTING PROCESS-RECORDS'
           DISPLAY 'WS-RECORD-COUNT = ' WS-RECORD-COUNT
           DISPLAY 'WS-CUSTOMER-ID  = [' WS-CUSTOMER-ID ']'
```

Wrapping values in brackets helps reveal trailing spaces or unexpected content in alphanumeric fields.

**Displaying numeric fields with context:**

```cobol
           DISPLAY 'BEFORE COMPUTE: WS-AMOUNT = '
                   WS-AMOUNT
                   ' WS-RATE = '
                   WS-RATE
           COMPUTE WS-RESULT = WS-AMOUNT * WS-RATE
           DISPLAY 'AFTER COMPUTE:  WS-RESULT = '
                   WS-RESULT
```

**Conditional debug displays:**

```cobol
       WORKING-STORAGE SECTION.
       01  WS-DEBUG-FLAG        PIC X VALUE 'N'.
           88 DEBUG-MODE        VALUE 'Y'.

       PROCEDURE DIVISION.
           IF DEBUG-MODE
               DISPLAY 'DEBUG: ENTERED PARA-1000'
               DISPLAY 'DEBUG: WS-INPUT = [' WS-INPUT ']'
           END-IF
```

This pattern allows you to turn debugging on and off by changing a single flag, or by passing it as a parameter or reading it from a control file.

**Displaying the current date/time for log correlation:**

```cobol
           ACCEPT WS-CURRENT-DATE FROM DATE YYYYMMDD
           ACCEPT WS-CURRENT-TIME FROM TIME
           DISPLAY WS-CURRENT-DATE ' ' WS-CURRENT-TIME
                   ' PROCESSING RECORD: ' WS-REC-KEY
```

**DISPLAY UPON CONSOLE vs. DISPLAY UPON SYSOUT:**

```cobol
           DISPLAY 'CRITICAL: FILE NOT FOUND' UPON CONSOLE
           DISPLAY 'DETAIL: PROCESSING ROW ' WS-ROW-NUM
               UPON SYSOUT
```

UPON CONSOLE typically writes to the operator console (high-visibility), while the default or UPON SYSOUT writes to the SYSOUT dataset assigned to the step.

### EXHIBIT Statement

EXHIBIT is a debugging verb available in some COBOL compilers. It automatically prints the variable name alongside its value, reducing the boilerplate of DISPLAY-based debugging.

```cobol
           EXHIBIT NAMED WS-COUNTER WS-TOTAL WS-STATUS
```

This produces output like:

```
WS-COUNTER = 00042
WS-TOTAL   = 00012345
WS-STATUS  = 00
```

**EXHIBIT variants:**

- `EXHIBIT NAMED` -- displays variable names and values.
- `EXHIBIT CHANGED` -- displays variables only when their values have changed since the last EXHIBIT CHANGED for that variable.
- `EXHIBIT CHANGED NAMED` -- combines both: shows name and value, but only on change.

```cobol
       PERFORM VARYING WS-IDX FROM 1 BY 1
           UNTIL WS-IDX > WS-MAX-ROWS
           EXHIBIT CHANGED NAMED WS-STATUS-CODE
           PERFORM PROCESS-ONE-ROW
       END-PERFORM
```

EXHIBIT CHANGED NAMED is particularly useful inside loops where you only want to see output when a status changes, avoiding thousands of duplicate lines.

Note: EXHIBIT is not part of the ANSI COBOL standard. It is a compiler extension. If portability is a concern, use DISPLAY instead.

### READY TRACE and RESET TRACE

READY TRACE activates paragraph-level tracing. When active, the runtime prints the name of each paragraph or section as it is entered.

```cobol
       PROCEDURE DIVISION.
       0000-MAIN.
           READY TRACE
           PERFORM 1000-INITIALIZE
           PERFORM 2000-PROCESS
           PERFORM 3000-FINALIZE
           RESET TRACE
           STOP RUN.
```

Sample trace output:

```
1000-INITIALIZE
1100-OPEN-FILES
1200-READ-CONTROL
2000-PROCESS
2100-READ-RECORD
2200-VALIDATE-RECORD
2100-READ-RECORD
2200-VALIDATE-RECORD
...
3000-FINALIZE
3100-CLOSE-FILES
```

**Selective tracing:**

```cobol
       2000-PROCESS.
           READY TRACE
           PERFORM 2100-READ-RECORD
           PERFORM 2200-VALIDATE-RECORD
           RESET TRACE
           PERFORM 2300-WRITE-OUTPUT.
```

You can bracket only the suspect code with READY TRACE / RESET TRACE to limit output volume.

**Caution:** READY TRACE produces enormous output in programs with tight loops. Always pair it with RESET TRACE and use it only around the area under investigation. Some installations disable TRACE at the system level for performance reasons.

### USE FOR DEBUGGING Declarative

The USE FOR DEBUGGING declarative is part of the COBOL standard debugging facility. It requires the WITH DEBUGGING MODE clause in the SOURCE-COMPUTER paragraph.

**Enabling debugging mode:**

```cobol
       ENVIRONMENT DIVISION.
       CONFIGURATION SECTION.
       SOURCE-COMPUTER. IBM-370 WITH DEBUGGING MODE.
```

**Declaring a debugging procedure:**

```cobol
       PROCEDURE DIVISION.
       DECLARATIVES.
       DEBUG-PARAGRAPHS SECTION.
           USE FOR DEBUGGING ON 2000-PROCESS.
       DEBUG-PARAGRAPHS-BODY.
           DISPLAY 'DEBUG: ENTERED 2000-PROCESS'
           DISPLAY 'DEBUG-ITEM = ' DEBUG-ITEM.
       END DECLARATIVES.
```

When `2000-PROCESS` is entered via PERFORM or fall-through, the debugging declarative fires automatically. The special register `DEBUG-ITEM` contains information about why the declarative was triggered.

**DEBUG-ITEM structure:**

| Field | Description |
|-------|-------------|
| DEBUG-LINE | Line number that caused the trigger |
| DEBUG-NAME | Name of the item being monitored |
| DEBUG-SUB-1, -2, -3 | Subscript values if applicable |
| DEBUG-CONTENTS | Contents of the identifier or status |

**Monitoring multiple items:**

```cobol
       DECLARATIVES.
       FILE-DEBUG SECTION.
           USE FOR DEBUGGING ON INPUT-FILE.
       FILE-DEBUG-BODY.
           DISPLAY 'FILE EVENT: ' DEBUG-NAME
                   ' LINE: ' DEBUG-LINE
                   ' CONTENTS: ' DEBUG-CONTENTS.

       PARA-DEBUG SECTION.
           USE FOR DEBUGGING ON ALL PROCEDURES.
       PARA-DEBUG-BODY.
           DISPLAY 'ENTERED: ' DEBUG-NAME.
       END DECLARATIVES.
```

`USE FOR DEBUGGING ON ALL PROCEDURES` monitors every paragraph and section entry -- effectively a built-in trace.

**Key point:** When WITH DEBUGGING MODE is removed from SOURCE-COMPUTER (or commented out), the compiler ignores all debugging declaratives and any lines with a "D" in column 7. This gives you a zero-overhead way to leave debugging code in production source without affecting performance.

### Debugging Declaratives

Beyond USE FOR DEBUGGING, the DECLARATIVES section also supports USE AFTER ERROR/EXCEPTION procedures for file I/O errors. See [error_handling.md](error_handling.md) for full coverage of USE AFTER STANDARD ERROR procedures, including syntax variants and interaction with inline error phrases.

### File Status Checking Patterns

Always define a file status variable for every file and check it after every I/O operation. This is the single most important defensive debugging pattern in COBOL file processing.

**Declaring file status:**

```cobol
       FILE-CONTROL.
           SELECT INPUT-FILE ASSIGN TO INFILE
               FILE STATUS IS WS-INPUT-STATUS.
           SELECT OUTPUT-FILE ASSIGN TO OUTFILE
               FILE STATUS IS WS-OUTPUT-STATUS.

       WORKING-STORAGE SECTION.
       01  WS-INPUT-STATUS     PIC XX VALUE SPACES.
       01  WS-OUTPUT-STATUS    PIC XX VALUE SPACES.
```

**Checking file status after every operation:**

```cobol
       1100-OPEN-FILES.
           OPEN INPUT INPUT-FILE
           IF WS-INPUT-STATUS NOT = '00'
               DISPLAY 'OPEN FAILED ON INPUT-FILE'
               DISPLAY 'FILE STATUS: ' WS-INPUT-STATUS
               MOVE 16 TO RETURN-CODE
               STOP RUN
           END-IF.

       2100-READ-RECORD.
           READ INPUT-FILE INTO WS-INPUT-RECORD
           EVALUATE WS-INPUT-STATUS
               WHEN '00'
                   CONTINUE
               WHEN '10'
                   SET END-OF-FILE TO TRUE
               WHEN OTHER
                   DISPLAY 'READ ERROR ON INPUT-FILE'
                   DISPLAY 'FILE STATUS: ' WS-INPUT-STATUS
                   MOVE 16 TO RETURN-CODE
                   STOP RUN
           END-EVALUATE.
```

See [file_handling.md](file_handling.md) for the complete file status code reference table.

## Common Patterns

### Defensive Coding Patterns

**Pattern 1: Validate numeric fields before arithmetic**

S0C7 data exceptions are the most common COBOL abend. Prevent them by validating data before use:

```cobol
       2200-VALIDATE-AMOUNT.
           IF WS-AMOUNT-IN IS NUMERIC
               MOVE WS-AMOUNT-IN TO WS-AMOUNT-NUM
               COMPUTE WS-RESULT = WS-AMOUNT-NUM * WS-RATE
           ELSE
               DISPLAY 'NON-NUMERIC AMOUNT: ['
                       WS-AMOUNT-IN ']'
               DISPLAY 'IN RECORD: ' WS-RECORD-KEY
               ADD 1 TO WS-ERROR-COUNT
           END-IF.
```

**Pattern 2: Subscript range checking**

Prevent S0C4 abends from out-of-range subscripts:

```cobol
           IF WS-INDEX >= 1 AND WS-INDEX <= 100
               MOVE WS-VALUE TO WS-TABLE-ENTRY(WS-INDEX)
           ELSE
               DISPLAY 'SUBSCRIPT OUT OF RANGE: ' WS-INDEX
               MOVE 16 TO RETURN-CODE
               STOP RUN
           END-IF
```

**Pattern 3: Check RETURN-CODE after CALL**

```cobol
           CALL 'SUBPROG1' USING WS-PARM-AREA
           EVALUATE RETURN-CODE
               WHEN 0
                   CONTINUE
               WHEN 4
                   DISPLAY 'WARNING FROM SUBPROG1'
               WHEN 8
                   DISPLAY 'ERROR FROM SUBPROG1, RC=' RETURN-CODE
                   PERFORM 9000-ABEND-ROUTINE
               WHEN OTHER
                   DISPLAY 'UNEXPECTED RC FROM SUBPROG1: '
                           RETURN-CODE
                   PERFORM 9000-ABEND-ROUTINE
           END-EVALUATE.
```

**Pattern 4: Centralized abend routine**

```cobol
       9000-ABEND-ROUTINE.
           DISPLAY '*** ABEND IN PROGRAM ' WS-PROGRAM-NAME
           DISPLAY '*** PARAGRAPH: ' WS-CURRENT-PARA
           DISPLAY '*** RECORD COUNT: ' WS-RECORD-COUNT
           DISPLAY '*** LAST KEY: ' WS-LAST-KEY
           DISPLAY '*** RETURN-CODE: ' RETURN-CODE
           MOVE 16 TO RETURN-CODE
           STOP RUN.
```

**Pattern 5: Tracking current paragraph for post-mortem**

```cobol
       WORKING-STORAGE SECTION.
       01  WS-CURRENT-PARA     PIC X(30) VALUE SPACES.

       PROCEDURE DIVISION.
       1000-INITIALIZE.
           MOVE '1000-INITIALIZE' TO WS-CURRENT-PARA
           ...
       2000-PROCESS.
           MOVE '2000-PROCESS' TO WS-CURRENT-PARA
           ...
```

If the program abends, WS-CURRENT-PARA in the dump shows which paragraph was executing. This is a low-cost alternative to READY TRACE.

**Pattern 6: Checkpoint displays for long-running batch**

```cobol
       2000-PROCESS.
           ADD 1 TO WS-RECORD-COUNT
           IF FUNCTION MOD(WS-RECORD-COUNT, 10000) = 0
               DISPLAY 'CHECKPOINT: ' WS-RECORD-COUNT
                       ' RECORDS PROCESSED AT '
                       FUNCTION CURRENT-DATE
           END-IF.
```

### Abend Code Reference Table

The following table covers the most frequently encountered system abend codes in COBOL programs:

| Code | Name | Common Cause | Typical Fix |
|------|------|-------------|-------------|
| **S0C1** | Operation Exception | Branching to non-executable storage; corrupted PERFORM stack; missing or invalid CALL target | Verify CALL program names; check for overwritten storage via buffer overflows; ensure PERFORM nesting is correct |
| **S0C4** | Protection Exception | Addressing storage the program does not own; subscript/index out of range; using an uninitialized pointer; LINKAGE SECTION mapping beyond passed data | Add subscript range checks; verify CALL parameters match the called program's expectations; check OCCURS DEPENDING ON values |
| **S0C7** | Data Exception | Performing arithmetic on a field that contains non-numeric data (spaces, low-values, alphanumeric characters in a PIC 9 field) | Use IS NUMERIC test before arithmetic; initialize numeric fields; inspect input data; check for uninitialized WORKING-STORAGE |
| **S0CB** | Decimal Divide Exception | Division by zero; or the quotient exceeds the receiving field size | Check divisor for zero before DIVIDE; ensure receiving field is large enough for the result |
| **S222** | Operator Cancel | Job was cancelled by the operator or an automated system | Not a program bug; check with operations; may indicate the job ran too long |
| **S322** | Time Limit Exceeded | CPU time exceeded the TIME parameter on the JOB or EXEC statement; program is in an infinite loop | Check for infinite loops (PERFORM UNTIL condition never true); increase TIME parameter if the workload is legitimately large |
| **S806** | Module Not Found | CALL target load module not found in any library in the search order (STEPLIB, JOBLIB, link list) | Verify module name spelling; check STEPLIB/JOBLIB DD; ensure module has been compiled, linked, and deployed |
| **S013** | Dataset Open Error | Conflicting DCB attributes; member not found in PDS; dataset not cataloged; DD statement missing or incorrect | Compare program FD to actual dataset attributes; check DD name matches SELECT ASSIGN; verify dataset exists in catalog |
| **S0C5** | Addressing Exception | Similar to S0C4; null pointer reference; referencing storage that has been freed | Check SET ADDRESS OF statements; verify LINKAGE SECTION pointers are assigned before use |
| **S0CA** | Decimal Overflow | Result of decimal arithmetic exceeds field capacity and ON SIZE ERROR was not specified or SIZE ERROR declarative is missing | Add ON SIZE ERROR clause; increase receiving field size |
| **U1026** | COBOL Sort Failed | SORT or MERGE operation failed; input or output procedure error; work dataset space issue | Check sort work DD space; verify input/output procedure logic; check file status |
| **U4038** | COBOL I/O Error | Unrecoverable I/O error in COBOL runtime | Check file status values; verify dataset allocation; check VSAM catalog |

### Reading Dumps and SYSUDUMP

When a COBOL program abends, the system can produce a storage dump. The level of detail depends on the DD statement provided:

- **SYSUDUMP** -- formatted dump of user storage (most common for COBOL debugging).
- **SYSABEND** -- formatted dump of user and system storage (larger, more detail).
- **SYSMDUMP** -- unformatted machine-readable dump (used with IPCS for analysis).

**JCL to request a dump:**

```
//SYSUDUMP DD SYSOUT=*
```

**How to read a SYSUDUMP:**

1. **Locate the abend code.** The dump header shows the system completion code (e.g., SYSTEM=0C7) and the PSW (Program Status Word).

2. **Find the PSW address.** The PSW contains the instruction address at the time of failure. Note the last six hex digits.

3. **Find the program entry point.** In the dump, look for the load module's entry point address in the save area trace or the Loader Information section.

4. **Calculate the offset.** Subtract the entry point address from the PSW address. The result is the offset into the program where the failure occurred.

5. **Cross-reference the compiler listing.** Open the compiler listing and find the offset in the assembler expansion or the condensed listing. This shows you the exact COBOL statement that failed.

6. **Examine storage.** Look at the data content at the addresses involved. For an S0C7, find the field that contained non-numeric data. The dump shows storage in hex -- look for X'F0' through X'F9' for valid zoned decimal, X'00' for low-values, or X'40' for spaces.

**Interpreting hex in dumps:**

| Hex Pattern | Meaning |
|-------------|---------|
| F0-F9 | Zoned decimal digits 0-9 |
| C0-C9 | Positive signed digits |
| D0-D9 | Negative signed digits |
| 40 | Space |
| 00 | Low-value / binary zero |
| FF | High-value |

**Tip:** Compiler options like LIST, MAP, and OFFSET produce the cross-reference and assembler listings you need for dump analysis. Always compile with these options in test environments.

### Compiler Diagnostics

The compiler listing is your primary compile-time debugging tool. Key sections include:

1. **Source listing** -- your COBOL source with line numbers.
2. **Cross-reference listing** (XREF option) -- shows every data item and every line where it is referenced, modified, or defined.
3. **Data Division map** (MAP option) -- shows the hex offset, length, and attributes of every data item in WORKING-STORAGE and LINKAGE SECTION.
4. **Condensed listing** (LIST or OFFSET option) -- maps COBOL statements to their hex offsets in the generated object code.
5. **Diagnostic messages** -- errors and warnings with severity, line number, and explanation.

**Using XREF to find unreferenced variables:**

The cross-reference listing flags data items that are defined but never referenced. These are candidates for cleanup and can sometimes indicate a bug (you meant to use the variable but referenced a different one).

**Using MAP to debug S0C7:**

When you know the offset of a failing instruction from a dump, the MAP listing tells you which WORKING-STORAGE variable occupies that offset. This directly identifies the corrupt field.

### Common Runtime Errors

Beyond the abend codes listed above, these runtime issues frequently affect COBOL programs:

**Infinite loops:** A PERFORM UNTIL whose condition is never satisfied. Symptoms: S322 abend or job hangs. Diagnose by adding checkpoint DISPLAYs inside the loop and verifying the loop variable is being modified correctly.

**Incorrect results without abend:** These are the hardest bugs. The program completes normally but produces wrong output. Causes include:
- Implicit truncation (moving a larger field to a smaller one without ON SIZE ERROR).
- Unsigned vs. signed arithmetic confusion.
- Incorrect REDEFINES causing data overlay.
- Wrong level numbers causing fields to be part of the wrong group.
- INITIALIZE resetting fields you did not expect (it initializes subordinate items based on their type).

**CALL parameter mismatches:** When the calling program and called program disagree on parameter size or type, storage can be corrupted silently. The called program may overlay storage beyond the passed parameter, causing S0C4 or data corruption detected much later.

**STRING/UNSTRING overflow:** Failing to check the POINTER value or OVERFLOW condition can cause truncation or incorrect parsing with no abend.

**PERFORM stack overflow:** Deeply nested or recursive PERFORMs can exhaust the PERFORM stack, causing S0C1 or S0C4. COBOL is not designed for recursion.

## Gotchas

- **DISPLAY changes timing.** Adding DISPLAY statements to a program changes its I/O behavior and timing. In rare cases, a bug that appears in production may not reproduce when DISPLAY statements are added (a COBOL version of the Heisenbug). Be aware that DISPLAY output is buffered and can affect program behavior in multi-step or CICS environments.

- **EXHIBIT is not portable.** EXHIBIT NAMED and EXHIBIT CHANGED are compiler extensions, not part of the ANSI or ISO COBOL standard. If your shop may ever migrate compilers, prefer DISPLAY-based debugging.

- **READY TRACE in a loop can fill SYSOUT.** A tight loop executing millions of times with TRACE active will produce gigabytes of output, potentially filling the spool and causing the job to fail with a different abend (e.g., S722). Always bracket TRACE narrowly.

- **WITH DEBUGGING MODE affects lines with "D" in column 7.** Any line with a "D" in column 7 is treated as a debugging line -- compiled when WITH DEBUGGING MODE is active, ignored otherwise. If someone accidentally places a "D" in column 7 of a production statement, removing DEBUGGING MODE will silently delete that logic.

- **Dumps require DD statements.** If the JCL does not include a SYSUDUMP, SYSABEND, or SYSMDUMP DD, no dump is produced on abend. Always include at least `//SYSUDUMP DD SYSOUT=*` in test JCL.

- **S0C7 location can be misleading.** The PSW points to the machine instruction that failed, but the COBOL statement may involve multiple machine instructions. A COMPUTE with several operands might fail on any one of them. Examine all operands, not just the receiving field.

- **File status "00" is not the only success.** For some operations, status "00" means success, but so does "02" (duplicate alternate key written or record read with duplicate key). Checking only for "00" may incorrectly flag a "02" as an error.

- **INITIALIZE does not set FILLER.** If you rely on INITIALIZE to clear a record area, FILLER items retain their previous values. This can leave stale data in output records.

- **RETURN-CODE is a shared register.** Every CALL resets RETURN-CODE. If you CALL a subroutine and then check RETURN-CODE after performing additional logic that includes another CALL, you are checking the wrong return code. Save RETURN-CODE immediately after each CALL.

- **Compiler options matter for debugging.** Options like SSRANGE (subscript range checking), TEST (debug symbol generation), and NOOPTIMIZE (prevent the optimizer from removing or reordering code) are essential for effective debugging but must be removed or changed for production to avoid performance impact.

- **ON SIZE ERROR has a cost.** Adding ON SIZE ERROR to every arithmetic operation is good for debugging but introduces overhead. In production, apply it selectively to operations where overflow is a genuine risk.

- **S806 can be a spelling error.** COBOL CALL literals are case-sensitive in the load module name. `CALL 'Subprog1'` and `CALL 'SUBPROG1'` may look for different modules depending on compiler options and link-edit settings.

## Related Topics

- **error_handling.md** -- Detailed coverage of ON SIZE ERROR, AT END, INVALID KEY, declarative USE AFTER ERROR procedures, and structured error handling patterns. Debugging and error handling are tightly coupled -- error handling prevents abends, and debugging diagnoses them when error handling is absent or insufficient.
- **data_types.md** -- Understanding PIC clauses, USAGE, and numeric representation is essential for diagnosing S0C7 abends and interpreting hex data in dumps.
- **file_handling.md** -- File status codes, OPEN/CLOSE/READ/WRITE mechanics, and VSAM error handling. Most debugging of file-related abends (S013, status 35, status 39) requires understanding the file handling model.
- **working_storage.md** -- Storage layout, VALUE clauses, INITIALIZE behavior, and REDEFINES. Many debugging scenarios involve unexpected storage content, which requires understanding how WORKING-STORAGE is allocated and initialized.
- **paragraph_flow.md** -- PERFORM mechanics, section vs. paragraph execution, and the PERFORM stack. Understanding flow control is critical for diagnosing S0C1 abends, infinite loops, and incorrect program flow.
