# I/O Processing

## Description
This file covers COBOL statements for accepting input, displaying output, and sorting/merging data: ACCEPT (FROM DATE/TIME/DAY/CONSOLE), DISPLAY (UPON CONSOLE/SYSOUT), SORT (with INPUT/OUTPUT PROCEDURE, USING/GIVING, sort keys), MERGE, RELEASE, RETURN, and the SORT-RETURN special register. Reference this file when working with console I/O, system date/time retrieval, or in-program sorting and merging of records.

## Table of Contents
- [Core Concepts](#core-concepts)
  - [ACCEPT Statement](#accept-statement)
  - [DISPLAY Statement](#display-statement)
  - [SORT Statement](#sort-statement)
  - [MERGE Statement](#merge-statement)
  - [RELEASE Statement](#release-statement)
  - [RETURN Statement](#return-statement)
  - [SORT-RETURN Special Register](#sort-return-special-register)
- [Syntax & Examples](#syntax--examples)
  - [ACCEPT Syntax](#accept-syntax)
  - [DISPLAY Syntax](#display-syntax)
  - [SORT Syntax](#sort-syntax)
  - [MERGE Syntax](#merge-syntax)
  - [RELEASE and RETURN Syntax](#release-and-return-syntax)
- [Common Patterns](#common-patterns)
- [Gotchas](#gotchas)
- [Related Topics](#related-topics)

## Core Concepts

### ACCEPT Statement

The ACCEPT statement retrieves data from sources outside the program: the operator console, the system date/time, environment variables, or an implementation-defined input device.

There are two broad forms:

1. **Identifier ACCEPT** -- reads a value from the default input device (typically SYSIN or the console) and moves it into a data item.
2. **Intrinsic-data ACCEPT** -- retrieves system date, time, day, or day-of-week values from the operating environment without any external I/O.

**Intrinsic-data sources:**

| Source           | Format            | Content                                      |
|------------------|-------------------|----------------------------------------------|
| `DATE`           | `PIC 9(6)` YYMMDD | Current date, two-digit year                 |
| `DATE YYYYMMDD`  | `PIC 9(8)`        | Current date, four-digit year                |
| `DAY`            | `PIC 9(5)` YYDDD  | Julian date, two-digit year                  |
| `DAY YYYYDDD`    | `PIC 9(7)`        | Julian date, four-digit year                 |
| `DAY-OF-WEEK`    | `PIC 9(1)`        | Day number: 1=Monday through 7=Sunday        |
| `TIME`           | `PIC 9(8)` HHMMSSss | Hours, minutes, seconds, hundredths       |

The intrinsic-data ACCEPT does not perform any I/O -- it reads from the system clock and is always available regardless of file assignments.

**Console / device ACCEPT:**

When ACCEPT is coded without a FROM clause (or with `FROM CONSOLE` / `FROM mnemonic-name`), the runtime reads from the operator console or from the system input stream (SYSIN). The data read is moved into the receiving identifier according to standard MOVE rules. If the input is shorter than the receiving field, it is padded; if longer, it is truncated.

On mainframe systems, `ACCEPT identifier` reads from the SYSIN DD by default. `ACCEPT identifier FROM CONSOLE` reads from the operator console (the CONSOLE device), which is often the MVS master console or TSO terminal.

### DISPLAY Statement

The DISPLAY statement sends data to the operator console or another output device. It is the simplest output mechanism in COBOL and is used primarily for:

- Writing messages to the job log or console
- Debugging output
- Writing short status or error messages
- Operator communication

**Target devices:**

| Clause              | Typical Destination                        |
|----------------------|--------------------------------------------|
| *(no UPON clause)*   | SYSOUT (default system output device)      |
| `UPON CONSOLE`       | Operator console (MVS master console, TSO) |
| `UPON SYSOUT`        | System output stream (SYSOUT DD)           |
| `UPON mnemonic-name` | Device mapped in SPECIAL-NAMES paragraph   |

DISPLAY converts all operands to their display (alphanumeric) representation and concatenates them left to right into a single output line. No spaces are inserted between operands unless they are explicitly included in the data or as literals.

The `WITH NO ADVANCING` option suppresses the trailing newline / carriage control, allowing the next DISPLAY or ACCEPT to continue on the same line.

### SORT Statement

The SORT statement arranges records in a specified order. COBOL sort is a powerful in-program facility that invokes the system sort utility (e.g., DFSORT, SyncSort on mainframes) or a built-in sort engine depending on the implementation.

**Key terminology:**

- **Sort file**: A file described with an `SD` (Sort Description) entry in the FILE SECTION. It serves as the work area for the sort operation. Unlike regular files, sort files are never explicitly opened or closed by the programmer.
- **Sort keys**: One or more fields within the sort record that determine the ordering. Each key is declared `ASCENDING` or `DESCENDING`. Multiple keys are evaluated left to right (major to minor).
- **USING**: Specifies one or more input files whose records are automatically read and fed to the sort.
- **GIVING**: Specifies one or more output files where sorted records are automatically written.
- **INPUT PROCEDURE**: Names a section or paragraph range that is performed to supply records to the sort via the RELEASE statement. This allows filtering, reformatting, or generating records before sorting.
- **OUTPUT PROCEDURE**: Names a section or paragraph range that is performed to retrieve sorted records via the RETURN statement. This allows processing, filtering, or reformatting records after sorting.

The sort operation has three phases:
1. **Input phase**: Records are supplied either by USING (automatic read from files) or by an INPUT PROCEDURE (manual feed via RELEASE).
2. **Sort phase**: Records are sorted by the specified keys. This is handled internally by the sort engine.
3. **Output phase**: Sorted records are delivered either by GIVING (automatic write to files) or by an OUTPUT PROCEDURE (manual retrieval via RETURN).

### MERGE Statement

The MERGE statement combines two or more identically sequenced files into a single ordered file. Unlike SORT, MERGE does not rearrange records -- it assumes each input file is already sorted on the specified merge keys and interleaves them to produce a single merged output.

MERGE requires:
- An `SD` file to serve as the merge work area
- Merge keys (ASCENDING or DESCENDING) on one or more fields
- A `USING` clause naming two or more input files (all must already be sorted on the merge keys)
- A `GIVING` clause or an `OUTPUT PROCEDURE` to handle the merged output

MERGE does **not** support an INPUT PROCEDURE. All input must come from files via USING.

### RELEASE Statement

The RELEASE statement sends a single record to the sort file during an INPUT PROCEDURE. It is analogous to a WRITE for the sort file. RELEASE can only be executed within the context of an INPUT PROCEDURE; executing it elsewhere is undefined and causes errors.

The syntax is:
- `RELEASE sort-record` -- releases the record area of the SD file as currently populated.
- `RELEASE sort-record FROM identifier` -- moves identifier to sort-record and then releases it.

### RETURN Statement

The RETURN statement retrieves the next record from a sort or merge file during an OUTPUT PROCEDURE. It is analogous to a READ for the sort file. RETURN can only be executed within the context of an OUTPUT PROCEDURE.

The RETURN statement sets the AT END condition when all sorted/merged records have been retrieved. The NOT AT END clause is available for processing each successfully returned record.

### SORT-RETURN Special Register

SORT-RETURN (also called SORT-STATUS in some implementations) is a special register automatically available in any program that uses SORT or MERGE. It is defined implicitly as `PIC S9(4) COMP` (a halfword binary value).

After a SORT or MERGE statement completes, SORT-RETURN contains the return code from the sort/merge operation:

| Value | Meaning                                     |
|-------|---------------------------------------------|
| 0     | Successful completion                       |
| 16    | Unsuccessful completion (sort/merge failed) |

Programs should test SORT-RETURN after each SORT or MERGE to verify success. Some implementations provide additional non-zero values for specific error conditions.

If the SORT or MERGE statement includes a GIVING file that cannot be opened, or the sort engine encounters an I/O error, SORT-RETURN will contain a non-zero value.

## Syntax & Examples

### ACCEPT Syntax

**Intrinsic-data ACCEPT (date/time):**

```cobol
       WORKING-STORAGE SECTION.
       01  WS-CURRENT-DATE        PIC 9(8).
       01  WS-CURRENT-DATE-R REDEFINES WS-CURRENT-DATE.
           05  WS-YEAR            PIC 9(4).
           05  WS-MONTH           PIC 9(2).
           05  WS-DAY             PIC 9(2).
       01  WS-CURRENT-TIME        PIC 9(8).
       01  WS-CURRENT-TIME-R REDEFINES WS-CURRENT-TIME.
           05  WS-HOURS           PIC 9(2).
           05  WS-MINUTES         PIC 9(2).
           05  WS-SECONDS         PIC 9(2).
           05  WS-HUNDREDTHS      PIC 9(2).
       01  WS-JULIAN-DATE         PIC 9(7).
       01  WS-DAY-OF-WEEK         PIC 9(1).

       PROCEDURE DIVISION.
           ACCEPT WS-CURRENT-DATE FROM DATE YYYYMMDD
           ACCEPT WS-CURRENT-TIME FROM TIME
           ACCEPT WS-JULIAN-DATE  FROM DAY YYYYDDD
           ACCEPT WS-DAY-OF-WEEK  FROM DAY-OF-WEEK
```

**Two-digit year form:**

```cobol
       01  WS-DATE-SHORT          PIC 9(6).
       01  WS-DAY-SHORT           PIC 9(5).

           ACCEPT WS-DATE-SHORT FROM DATE
      *>   WS-DATE-SHORT now contains YYMMDD
           ACCEPT WS-DAY-SHORT FROM DAY
      *>   WS-DAY-SHORT now contains YYDDD
```

**Console ACCEPT:**

```cobol
       01  WS-USER-INPUT          PIC X(80).

           DISPLAY 'ENTER CUSTOMER ID: ' WITH NO ADVANCING
           ACCEPT WS-USER-INPUT FROM CONSOLE

      *>   Default device (typically SYSIN):
           ACCEPT WS-USER-INPUT
```

### DISPLAY Syntax

**Basic DISPLAY:**

```cobol
           DISPLAY 'PROGRAM ABC123 STARTED'
           DISPLAY 'RECORDS PROCESSED: ' WS-REC-COUNT
           DISPLAY 'DATE: ' WS-YEAR '/' WS-MONTH '/' WS-DAY
```

**DISPLAY with device targeting:**

```cobol
           DISPLAY 'OPERATOR: CONFIRM TAPE MOUNT'
               UPON CONSOLE
           DISPLAY 'BATCH LOG MESSAGE'
               UPON SYSOUT
```

**WITH NO ADVANCING:**

```cobol
           DISPLAY 'PROCESSING: ' WITH NO ADVANCING
           DISPLAY WS-FILENAME
      *>   Output appears on a single line:
      *>   PROCESSING: CUSTOMER.DAT
```

**Multiple operands -- concatenation with no automatic spacing:**

```cobol
           DISPLAY WS-LAST-NAME ', ' WS-FIRST-NAME
               ' -- ACCOUNT: ' WS-ACCT-NUM
      *>   Produces: SMITH, JOHN -- ACCOUNT: 12345
```

### SORT Syntax

**Simple SORT with USING/GIVING:**

```cobol
       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT SORT-FILE   ASSIGN TO SORTWORK.
           SELECT INPUT-FILE  ASSIGN TO INFILE.
           SELECT OUTPUT-FILE ASSIGN TO OUTFILE.

       DATA DIVISION.
       FILE SECTION.
       SD  SORT-FILE.
       01  SORT-RECORD.
           05  SORT-KEY-DEPT      PIC X(4).
           05  SORT-KEY-EMPNO     PIC 9(6).
           05  SORT-DATA          PIC X(70).

       FD  INPUT-FILE.
       01  INPUT-RECORD           PIC X(80).

       FD  OUTPUT-FILE.
       01  OUTPUT-RECORD          PIC X(80).

       PROCEDURE DIVISION.
           SORT SORT-FILE
               ON ASCENDING KEY SORT-KEY-DEPT
               ON DESCENDING KEY SORT-KEY-EMPNO
               USING INPUT-FILE
               GIVING OUTPUT-FILE
           IF SORT-RETURN NOT = ZERO
               DISPLAY 'SORT FAILED, RC=' SORT-RETURN
               STOP RUN
           END-IF
           STOP RUN.
```

**SORT with INPUT PROCEDURE and OUTPUT PROCEDURE:**

```cobol
       PROCEDURE DIVISION.
       0000-MAIN.
           SORT SORT-FILE
               ON ASCENDING KEY SORT-KEY-DEPT
               ON ASCENDING KEY SORT-KEY-EMPNO
               INPUT PROCEDURE IS 1000-SELECT-RECORDS
               OUTPUT PROCEDURE IS 2000-WRITE-RESULTS
           IF SORT-RETURN NOT = ZERO
               DISPLAY 'SORT FAILED, RC=' SORT-RETURN
               MOVE 16 TO RETURN-CODE
               STOP RUN
           END-IF
           STOP RUN.

       1000-SELECT-RECORDS.
           OPEN INPUT INPUT-FILE
           PERFORM UNTIL WS-EOF-FLAG = 'Y'
               READ INPUT-FILE INTO WS-INPUT-REC
                   AT END
                       MOVE 'Y' TO WS-EOF-FLAG
                   NOT AT END
                       IF WS-INPUT-DEPT NOT = 'TERM'
                           MOVE WS-INPUT-REC TO SORT-RECORD
                           RELEASE SORT-RECORD
                       END-IF
               END-READ
           END-PERFORM
           CLOSE INPUT-FILE.

       2000-WRITE-RESULTS.
           OPEN OUTPUT OUTPUT-FILE
           PERFORM UNTIL WS-SORT-EOF = 'Y'
               RETURN SORT-FILE INTO WS-OUTPUT-REC
                   AT END
                       MOVE 'Y' TO WS-SORT-EOF
                   NOT AT END
                       WRITE OUTPUT-RECORD
                           FROM WS-OUTPUT-REC
               END-RETURN
           END-PERFORM
           CLOSE OUTPUT-FILE.
```

**SORT with multiple input files via USING:**

```cobol
           SORT SORT-FILE
               ON ASCENDING KEY SORT-KEY-REGION
               ON ASCENDING KEY SORT-KEY-CUSTID
               USING FILE-EAST FILE-WEST FILE-CENTRAL
               GIVING MASTER-FILE
```

**SORT with DUPLICATES IN ORDER:**

```cobol
           SORT SORT-FILE
               ON ASCENDING KEY SORT-KEY-DATE
               WITH DUPLICATES IN ORDER
               USING TRANS-FILE
               GIVING SORTED-TRANS
      *>   Records with equal keys retain their original
      *>   input sequence.
```

### MERGE Syntax

```cobol
       FILE SECTION.
       SD  MERGE-FILE.
       01  MERGE-RECORD.
           05  MERGE-KEY-DATE     PIC 9(8).
           05  MERGE-KEY-SEQ      PIC 9(4).
           05  MERGE-DATA         PIC X(68).

       FD  REGION-EAST-FILE.
       01  REGION-EAST-REC        PIC X(80).

       FD  REGION-WEST-FILE.
       01  REGION-WEST-REC        PIC X(80).

       FD  COMBINED-FILE.
       01  COMBINED-REC           PIC X(80).

       PROCEDURE DIVISION.
           MERGE MERGE-FILE
               ON ASCENDING KEY MERGE-KEY-DATE
               ON ASCENDING KEY MERGE-KEY-SEQ
               USING REGION-EAST-FILE
                     REGION-WEST-FILE
               GIVING COMBINED-FILE
           IF SORT-RETURN NOT = ZERO
               DISPLAY 'MERGE FAILED, RC=' SORT-RETURN
               STOP RUN
           END-IF
           STOP RUN.
```

**MERGE with OUTPUT PROCEDURE:**

```cobol
           MERGE MERGE-FILE
               ON ASCENDING KEY MERGE-KEY-DATE
               USING DAILY-FILE-1
                     DAILY-FILE-2
                     DAILY-FILE-3
               OUTPUT PROCEDURE IS 3000-PROCESS-MERGED

       3000-PROCESS-MERGED.
           OPEN OUTPUT REPORT-FILE
           MOVE ZERO TO WS-TOTAL-AMT
           PERFORM UNTIL WS-MERGE-EOF = 'Y'
               RETURN MERGE-FILE INTO WS-MERGED-REC
                   AT END
                       MOVE 'Y' TO WS-MERGE-EOF
                   NOT AT END
                       ADD WS-MERGED-AMT TO WS-TOTAL-AMT
                       PERFORM 3100-FORMAT-REPORT-LINE
                       WRITE REPORT-RECORD
                           FROM WS-REPORT-LINE
               END-RETURN
           END-PERFORM
           CLOSE REPORT-FILE.
```

### RELEASE and RETURN Syntax

**RELEASE (within INPUT PROCEDURE):**

```cobol
      *>   Form 1: Release the sort record directly
           MOVE WS-INPUT-REC TO SORT-RECORD
           RELEASE SORT-RECORD

      *>   Form 2: Release FROM another data item
           RELEASE SORT-RECORD FROM WS-INPUT-REC
      *>   This is equivalent to:
      *>     MOVE WS-INPUT-REC TO SORT-RECORD
      *>     RELEASE SORT-RECORD
```

**RETURN (within OUTPUT PROCEDURE):**

```cobol
      *>   Form 1: Return into the sort record area
           RETURN SORT-FILE
               AT END
                   SET WS-END-OF-SORT TO TRUE
               NOT AT END
                   MOVE SORT-RECORD TO WS-OUTPUT-REC
           END-RETURN

      *>   Form 2: Return INTO another data item
           RETURN SORT-FILE INTO WS-OUTPUT-REC
               AT END
                   SET WS-END-OF-SORT TO TRUE
               NOT AT END
                   PERFORM 5000-PROCESS-RECORD
           END-RETURN
```

## Common Patterns

### Timestamp Logging

A very common pattern is capturing the current date and time at program start for logging and audit trails:

```cobol
       01  WS-TIMESTAMP.
           05  WS-TS-DATE         PIC 9(8).
           05  WS-TS-TIME         PIC 9(8).
       01  WS-FORMATTED-TS        PIC X(23).

       0100-GET-TIMESTAMP.
           ACCEPT WS-TS-DATE FROM DATE YYYYMMDD
           ACCEPT WS-TS-TIME FROM TIME
           STRING WS-TS-DATE(1:4)  '-'
                  WS-TS-DATE(5:2)  '-'
                  WS-TS-DATE(7:2)  ' '
                  WS-TS-TIME(1:2)  ':'
                  WS-TS-TIME(3:2)  ':'
                  WS-TS-TIME(5:2)  '.'
                  WS-TS-TIME(7:2)
               DELIMITED BY SIZE
               INTO WS-FORMATTED-TS
           END-STRING.
      *>   Result: 2025-03-15 14:30:22.05
```

### Sort with Record Selection (Filter Before Sort)

A common mainframe batch pattern is to read a large file, filter out unwanted records, and sort only the qualifying ones. This is more efficient than sorting everything and discarding afterward:

```cobol
       1000-FILTER-AND-RELEASE.
           OPEN INPUT TRANSACTION-FILE
           PERFORM UNTIL WS-EOF = 'Y'
               READ TRANSACTION-FILE INTO WS-TRANS-REC
                   AT END
                       MOVE 'Y' TO WS-EOF
                   NOT AT END
                       EVALUATE TRUE
                           WHEN WS-TRANS-TYPE = 'A'
                           WHEN WS-TRANS-TYPE = 'C'
                               MOVE WS-TRANS-REC
                                   TO SORT-RECORD
                               RELEASE SORT-RECORD
                           WHEN OTHER
                               ADD 1 TO WS-SKIPPED-COUNT
                       END-EVALUATE
               END-READ
           END-PERFORM
           CLOSE TRANSACTION-FILE.
```

### Sort with Control Break Reporting

An OUTPUT PROCEDURE can apply control break logic to sorted records as they are returned from the sort. The RETURN statement replaces READ, and the AT END condition signals all sorted records have been consumed. Within this context, the control break pattern (detecting key changes, printing subtotals, resetting accumulators) operates the same as in standard sequential processing. See [batch_patterns.md](batch_patterns.md) for full coverage of single-level and multi-level control break logic, including accumulator management and final-break handling.

### Elapsed-Time Calculation

Capturing time at the start and end of a batch step to report elapsed duration:

```cobol
       01  WS-START-TIME          PIC 9(8).
       01  WS-END-TIME            PIC 9(8).
       01  WS-START-SECS          PIC 9(7)V99.
       01  WS-END-SECS            PIC 9(7)V99.
       01  WS-ELAPSED-SECS        PIC 9(7)V99.

       9000-COMPUTE-ELAPSED.
           COMPUTE WS-START-SECS =
               WS-START-TIME(1:2) * 3600
             + WS-START-TIME(3:2) * 60
             + WS-START-TIME(5:2)
             + WS-START-TIME(7:2) / 100
           COMPUTE WS-END-SECS =
               WS-END-TIME(1:2) * 3600
             + WS-END-TIME(3:2) * 60
             + WS-END-TIME(5:2)
             + WS-END-TIME(7:2) / 100
           COMPUTE WS-ELAPSED-SECS =
               WS-END-SECS - WS-START-SECS
           DISPLAY 'ELAPSED TIME: '
               WS-ELAPSED-SECS ' SECONDS'.
```

### Multi-File Merge for Daily Consolidation

A typical batch pattern where regional daily files are merged into a single master:

```cobol
           MERGE WORK-FILE
               ON ASCENDING KEY WF-ACCOUNT-NUM
               ON ASCENDING KEY WF-TRANS-DATE
               USING REGION-1-DAILY
                     REGION-2-DAILY
                     REGION-3-DAILY
                     REGION-4-DAILY
               GIVING CONSOLIDATED-DAILY
```

### Operator Communication Pattern

Prompting for and accepting operator decisions during batch execution:

```cobol
       8000-OPERATOR-CONFIRM.
           DISPLAY 'WXY001I - RECORDS TO PURGE: '
               WS-PURGE-COUNT UPON CONSOLE
           DISPLAY 'WXY002A - REPLY Y TO CONTINUE, '
               'N TO ABORT' UPON CONSOLE
           ACCEPT WS-REPLY FROM CONSOLE
           IF WS-REPLY = 'Y' OR 'y'
               CONTINUE
           ELSE
               DISPLAY 'WXY003I - PURGE ABORTED BY '
                   'OPERATOR' UPON CONSOLE
               MOVE 8 TO RETURN-CODE
               STOP RUN
           END-IF.
```

### SORT-RETURN Defensive Pattern

Always checking SORT-RETURN after sort operations for robust error handling:

```cobol
           SORT SORT-FILE
               ON ASCENDING KEY SORT-KEY
               USING INPUT-FILE
               GIVING OUTPUT-FILE

           EVALUATE SORT-RETURN
               WHEN ZERO
                   DISPLAY 'SORT COMPLETED SUCCESSFULLY'
               WHEN 16
                   DISPLAY 'SORT FAILED - RC=16'
                   MOVE 16 TO RETURN-CODE
                   PERFORM 9999-ABEND-ROUTINE
               WHEN OTHER
                   DISPLAY 'SORT WARNING - RC='
                       SORT-RETURN
           END-EVALUATE.
```

## Gotchas

- **ACCEPT FROM DATE gives a two-digit year.** Using `ACCEPT ws-date FROM DATE` returns YYMMDD, not YYYYMMDD. This is a year-2000 hazard. Always use `ACCEPT ws-date FROM DATE YYYYMMDD` for four-digit years. The same applies to `FROM DAY` vs `FROM DAY YYYYDDD`.

- **ACCEPT FROM TIME returns hundredths, not milliseconds.** The last two digits of the TIME value are hundredths of a second (00-99), not milliseconds. Programs that treat them as milliseconds will miscalculate durations.

- **DISPLAY concatenates without spaces.** Multiple operands in a DISPLAY are concatenated directly. `DISPLAY WS-NAME WS-AMOUNT` produces `SMITH00125000`, not `SMITH 00125000`. Always include explicit space literals or pad fields appropriately.

- **DISPLAY of numeric items shows the internal representation.** A `PIC S9(5) COMP-3` field displayed directly may show unexpected characters because DISPLAY converts from the internal format. Move numeric items to edited or alphanumeric fields before displaying them for readable output.

- **Sort files (SD) must never be explicitly opened or closed.** Issuing OPEN or CLOSE on an SD file causes a runtime error. The sort engine manages the sort file's lifecycle internally.

- **RELEASE is only valid inside an INPUT PROCEDURE.** Executing RELEASE outside the scope of the INPUT PROCEDURE named in a SORT statement is undefined behavior and typically causes an abend. The same restriction applies to RETURN -- it is only valid inside an OUTPUT PROCEDURE.

- **INPUT PROCEDURE and USING are mutually exclusive.** You cannot specify both `INPUT PROCEDURE` and `USING` in the same SORT statement. Likewise, `OUTPUT PROCEDURE` and `GIVING` are mutually exclusive. Attempting to mix them causes a compile-time error.

- **MERGE does not support INPUT PROCEDURE.** Unlike SORT, the MERGE statement only accepts USING for input. If you need to filter or transform records before merging, you must pre-process the input files separately.

- **Sort key order matters: left to right = major to minor.** The first ON ASCENDING/DESCENDING KEY clause specifies the primary (most significant) sort key. Subsequent clauses specify progressively minor keys. Reversing the order produces entirely different results.

- **Forgetting to check SORT-RETURN.** If the sort utility fails (e.g., insufficient sort work space, I/O error on USING files), the program continues executing after the SORT statement. Without testing SORT-RETURN, the program may process empty or incomplete output, producing corrupt results silently.

- **DUPLICATES IN ORDER is not the default.** Without `WITH DUPLICATES IN ORDER`, the sort engine may return records with equal keys in any order. If the sequence of duplicate-key records matters (e.g., transactions for the same account must remain in arrival order), you must specify `WITH DUPLICATES IN ORDER`.

- **ACCEPT FROM CONSOLE in batch can block the job.** On mainframe systems, `ACCEPT FROM CONSOLE` waits for an operator reply via the MVS console. In unattended batch jobs, this can cause the job to hang indefinitely waiting for input that never arrives. Use ACCEPT from SYSIN for non-interactive batch input.

- **SORT work space (SORTWK DDs) must be allocated.** On mainframe systems, the SORT statement requires sort work data sets (typically SORTWK01 through SORTWKnn) to be allocated in the JCL. Missing or undersized sort work allocations cause the sort to fail with a non-zero SORT-RETURN. Some sort products support HIPERSPACE or memory-only sorting, but work files are the safe default.

- **GIVING opens the output file for you.** When using GIVING, the sort engine opens and closes the output file automatically. If the program has already opened that file, or attempts to open it afterward without accounting for this, unexpected results or errors occur. The same applies to USING -- the sort engine opens and closes the input files.

- **Midnight crossover in elapsed-time calculations.** If a program spans midnight, `ACCEPT FROM TIME` at the end will return a smaller value than at the start. Elapsed-time calculations that do not account for this wraparound will produce negative or nonsensical results.

## Related Topics

- **file_handling.md** -- SORT and MERGE operate on files defined with FD/SD entries. File handling covers the FD-level file I/O (OPEN, CLOSE, READ, WRITE) that is used alongside INPUT/OUTPUT PROCEDUREs and by the USING/GIVING clauses.
- **jcl_interaction.md** -- On mainframe systems, SORT requires SORTWK DD statements and the input/output files referenced by USING/GIVING must be allocated via JCL DD statements. ACCEPT/DISPLAY behavior is also influenced by SYSIN/SYSOUT DD allocation.
- **batch_patterns.md** -- Sort with control breaks, record filtering via INPUT PROCEDURE, and operator communication via ACCEPT/DISPLAY are fundamental batch processing patterns.
- **working_storage.md** -- The data items used as targets for ACCEPT and as sources for DISPLAY are defined in WORKING-STORAGE. Sort key fields and record layouts referenced by RELEASE/RETURN are also defined there or in the FILE SECTION.
