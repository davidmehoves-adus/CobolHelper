# Batch Patterns

## Description
Covers the dominant design patterns used in COBOL batch programs: sequential file processing, control break logic, master-transaction file matching, report generation, checkpoint/restart, multi-file processing, end-of-file handling, accumulator patterns, and header/footer/detail record processing. Reference this file when designing, reviewing, or debugging any batch COBOL program that reads input files, produces reports, or updates master files.

## Table of Contents
- [Core Concepts](#core-concepts)
  - [What Makes Batch Different](#what-makes-batch-different)
  - [The Canonical Batch Loop](#the-canonical-batch-loop)
  - [End-of-File Handling](#end-of-file-handling)
  - [Accumulator Patterns](#accumulator-patterns)
- [Syntax & Examples](#syntax--examples)
  - [Sequential File Processing](#sequential-file-processing)
  - [Control Break Logic](#control-break-logic)
  - [Master-Transaction File Matching](#master-transaction-file-matching)
  - [Report Generation Patterns](#report-generation-patterns)
  - [Checkpoint/Restart](#checkpointrestart)
  - [Multi-File Processing](#multi-file-processing)
  - [Header/Footer/Detail Record Processing](#headerfooterdetail-record-processing)
- [Common Patterns](#common-patterns)
- [Gotchas](#gotchas)
- [Related Topics](#related-topics)

## Core Concepts

### What Makes Batch Different

Batch programs process data in bulk without human interaction. They typically run on a schedule (nightly, weekly, monthly), read one or more input files, apply business logic, and produce output files or reports. The program lifecycle is: open files, process all records in a loop, close files, stop run.

Batch programs are judged by correctness, throughput, and restartability. A batch job that abends at record 999,000 of 1,000,000 must not require reprocessing from the beginning --- hence the importance of checkpoint/restart.

### The Canonical Batch Loop

Nearly every COBOL batch program follows this skeleton:

1. **Initialization** -- open files, initialize accumulators, read the first record (the "priming read").
2. **Main loop** -- process records until end-of-file. Each iteration ends with the next read.
3. **Termination** -- write final totals, close files, stop run.

The priming read is essential. It ensures the AT END condition is tested before entering the loop body, which correctly handles the edge case of an empty input file.

### End-of-File Handling

COBOL signals end-of-file through the `AT END` clause on the `READ` statement or through the `FILE STATUS` field. Two common strategies:

1. **Flag-based** -- a level-88 condition set inside `AT END`, tested in a `PERFORM UNTIL`.
2. **File-status-based** -- the program checks the two-byte file status field after every I/O operation. Status `10` means end-of-file for sequential reads.

The flag-based approach is simpler for single-file programs. The file-status approach is more robust and essential when a program processes multiple input files, because it lets the program distinguish between a normal EOF and an I/O error.

```cobol
       WORKING-STORAGE SECTION.
       01  WS-EOF-FLAG          PIC X VALUE 'N'.
           88 END-OF-FILE                VALUE 'Y'.
           88 NOT-END-OF-FILE            VALUE 'N'.

       01  WS-FILE-STATUS        PIC XX.
           88 FS-SUCCESS                 VALUE '00'.
           88 FS-END-OF-FILE             VALUE '10'.
```

### Accumulator Patterns

Accumulators are numeric fields that tally counts, sums, or hashes as records are processed. They serve three purposes:

1. **Control totals** -- verify that every input record was processed (record count, hash total on a key field).
2. **Report totals** -- subtotals and grand totals printed on reports.
3. **Balancing** -- debits must equal credits; input count must equal output count plus reject count.

Accumulators must be initialized before the loop (typically via `INITIALIZE` or explicit `MOVE ZEROS`). They are rolled up at control breaks and printed at termination.

```cobol
       01  WS-ACCUMULATORS.
           05  WS-RECORD-COUNT   PIC 9(07) VALUE ZEROS.
           05  WS-TOTAL-AMOUNT   PIC S9(11)V99 VALUE ZEROS.
           05  WS-HASH-TOTAL     PIC 9(11) VALUE ZEROS.
           05  WS-ERROR-COUNT    PIC 9(05) VALUE ZEROS.
```

A common discipline is to keep a separate set of accumulators for each control-break level (detail, minor, intermediate, major, grand) so that rolling up is straightforward.

## Syntax & Examples

### Sequential File Processing

The most fundamental batch pattern: read every record from an input file, process it, and write to an output file.

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. SEQ-PROCESS.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT INPUT-FILE  ASSIGN TO INFILE
               FILE STATUS IS WS-IN-STATUS.
           SELECT OUTPUT-FILE ASSIGN TO OUTFILE
               FILE STATUS IS WS-OUT-STATUS.

       DATA DIVISION.
       FILE SECTION.
       FD  INPUT-FILE.
       01  INPUT-RECORD         PIC X(80).

       FD  OUTPUT-FILE.
       01  OUTPUT-RECORD        PIC X(80).

       WORKING-STORAGE SECTION.
       01  WS-IN-STATUS         PIC XX.
       01  WS-OUT-STATUS        PIC XX.
       01  WS-EOF-FLAG          PIC X VALUE 'N'.
           88 END-OF-FILE               VALUE 'Y'.
           88 NOT-END-OF-FILE           VALUE 'N'.

       PROCEDURE DIVISION.
       0000-MAIN.
           PERFORM 1000-INITIALIZE
           PERFORM 2000-PROCESS
               UNTIL END-OF-FILE
           PERFORM 3000-TERMINATE
           STOP RUN
           .

       1000-INITIALIZE.
           OPEN INPUT  INPUT-FILE
           OPEN OUTPUT OUTPUT-FILE
           PERFORM 9000-READ-INPUT
           .

       2000-PROCESS.
      *    -- business logic here --
           MOVE INPUT-RECORD TO OUTPUT-RECORD
           WRITE OUTPUT-RECORD
           PERFORM 9000-READ-INPUT
           .

       3000-TERMINATE.
           CLOSE INPUT-FILE
           CLOSE OUTPUT-FILE
           .

       9000-READ-INPUT.
           READ INPUT-FILE INTO INPUT-RECORD
               AT END
                   SET END-OF-FILE TO TRUE
           END-READ
           .
```

Key points: the priming read is in `1000-INITIALIZE`, the trailing read is at the end of `2000-PROCESS`, and `0000-MAIN` drives the loop.

### Control Break Logic

A control break occurs when a key field changes between consecutive records. The input file must be sorted by the control field(s). Control breaks trigger subtotal printing and accumulator resets.

**Single-level control break:**

```cobol
       WORKING-STORAGE SECTION.
       01  WS-PREV-DEPT         PIC X(04) VALUE SPACES.
       01  WS-DEPT-TOTAL        PIC S9(09)V99 VALUE ZEROS.
       01  WS-GRAND-TOTAL       PIC S9(11)V99 VALUE ZEROS.

       PROCEDURE DIVISION.
       2000-PROCESS.
           IF DEPT-CODE OF INPUT-RECORD
               NOT = WS-PREV-DEPT
               AND WS-PREV-DEPT NOT = SPACES
               PERFORM 2500-DEPT-BREAK
           END-IF

           ADD AMOUNT OF INPUT-RECORD TO WS-DEPT-TOTAL
           MOVE DEPT-CODE OF INPUT-RECORD TO WS-PREV-DEPT

           PERFORM 9000-READ-INPUT
           .

       2500-DEPT-BREAK.
      *    Print subtotal for previous department
           MOVE WS-PREV-DEPT  TO RPT-DEPT
           MOVE WS-DEPT-TOTAL TO RPT-TOTAL
           WRITE REPORT-RECORD FROM DEPT-TOTAL-LINE
      *    Roll up and reset
           ADD WS-DEPT-TOTAL TO WS-GRAND-TOTAL
           MOVE ZEROS TO WS-DEPT-TOTAL
           .
```

**Multi-level control break (major/intermediate/minor):**

When multiple control levels exist, breaks cascade from minor to major. A change in the major key forces breaks at all lower levels first.

```cobol
       2000-PROCESS.
      *    Check breaks from major to minor
           EVALUATE TRUE
               WHEN REGION OF INPUT-RECORD
                   NOT = WS-PREV-REGION
                   PERFORM 2100-MINOR-BREAK
                   PERFORM 2200-INTERMEDIATE-BREAK
                   PERFORM 2300-MAJOR-BREAK

               WHEN BRANCH OF INPUT-RECORD
                   NOT = WS-PREV-BRANCH
                   PERFORM 2100-MINOR-BREAK
                   PERFORM 2200-INTERMEDIATE-BREAK

               WHEN DEPT OF INPUT-RECORD
                   NOT = WS-PREV-DEPT
                   PERFORM 2100-MINOR-BREAK
           END-EVALUATE

           ADD AMOUNT OF INPUT-RECORD TO WS-DEPT-TOTAL
           MOVE DEPT   OF INPUT-RECORD TO WS-PREV-DEPT
           MOVE BRANCH OF INPUT-RECORD TO WS-PREV-BRANCH
           MOVE REGION OF INPUT-RECORD TO WS-PREV-REGION

           PERFORM 9000-READ-INPUT
           .
```

The termination paragraph must also force a final break at all levels to flush the last group's totals.

```cobol
       3000-TERMINATE.
           IF WS-PREV-DEPT NOT = SPACES
               PERFORM 2100-MINOR-BREAK
               PERFORM 2200-INTERMEDIATE-BREAK
               PERFORM 2300-MAJOR-BREAK
           END-IF
           PERFORM 2400-GRAND-TOTAL
           CLOSE INPUT-FILE
           CLOSE REPORT-FILE
           .
```

### Master-Transaction File Matching

This is the classic "balance-line" or "matching" algorithm. Both files are sorted on the same key. The program compares keys and takes one of three actions:

- **Master key < Transaction key** -- unmatched master (no transactions for this master). Write master unchanged, read next master.
- **Master key = Transaction key** -- matched. Apply transaction to master, read next transaction.
- **Master key > Transaction key** -- unmatched transaction (new record or error). Handle accordingly, read next transaction.

```cobol
       WORKING-STORAGE SECTION.
       01  WS-MASTER-EOF        PIC X VALUE 'N'.
           88 MASTER-EOF                VALUE 'Y'.
       01  WS-TRANS-EOF         PIC X VALUE 'N'.
           88 TRANS-EOF                 VALUE 'Y'.
       01  HIGH-VALUE-KEY       PIC X(10) VALUE HIGH-VALUES.

       PROCEDURE DIVISION.
       0000-MAIN.
           PERFORM 1000-INITIALIZE
           PERFORM 2000-PROCESS
               UNTIL MASTER-EOF AND TRANS-EOF
           PERFORM 3000-TERMINATE
           STOP RUN
           .

       1000-INITIALIZE.
           OPEN INPUT  MASTER-FILE
           OPEN INPUT  TRANS-FILE
           OPEN OUTPUT NEW-MASTER-FILE
           PERFORM 9100-READ-MASTER
           PERFORM 9200-READ-TRANS
           .

       2000-PROCESS.
           EVALUATE TRUE
               WHEN MASTER-KEY < TRANS-KEY
      *            Unmatched master -- write unchanged
                   WRITE NEW-MASTER-REC FROM MASTER-RECORD
                   PERFORM 9100-READ-MASTER

               WHEN MASTER-KEY = TRANS-KEY
      *            Match -- apply update
                   PERFORM 2500-APPLY-TRANSACTION
                   PERFORM 9200-READ-TRANS

               WHEN MASTER-KEY > TRANS-KEY
      *            Unmatched transaction -- add or error
                   PERFORM 2600-HANDLE-NEW-TRANS
                   PERFORM 9200-READ-TRANS
           END-EVALUATE
           .

       9100-READ-MASTER.
           READ MASTER-FILE INTO MASTER-RECORD
               AT END
                   SET MASTER-EOF TO TRUE
                   MOVE HIGH-VALUES TO MASTER-KEY
           END-READ
           .

       9200-READ-TRANS.
           READ TRANS-FILE INTO TRANS-RECORD
               AT END
                   SET TRANS-EOF TO TRUE
                   MOVE HIGH-VALUES TO TRANS-KEY
           END-READ
           .
```

The `HIGH-VALUES` trick is critical. When one file reaches EOF, its key is set to the highest possible value, which forces the other file to drain completely before the `UNTIL` condition becomes true.

### Report Generation Patterns

Report programs combine sequential processing, control breaks, and page-overflow handling.

**Page overflow and line counting:**

```cobol
       WORKING-STORAGE SECTION.
       01  WS-LINE-COUNT        PIC 99 VALUE 99.
       01  WS-PAGE-COUNT        PIC 9(04) VALUE ZEROS.
       01  WS-LINES-PER-PAGE    PIC 99 VALUE 55.

       01  RPT-HEADER-1.
           05  FILLER            PIC X(30) VALUE
               'MONTHLY SALES REPORT'.
           05  FILLER            PIC X(20) VALUE SPACES.
           05  RPT-DATE          PIC X(10).
           05  FILLER            PIC X(06) VALUE ' PAGE '.
           05  RPT-PAGE          PIC ZZZ9.

       01  RPT-DETAIL-LINE.
           05  RPT-DEPT          PIC X(04).
           05  FILLER            PIC X(03) VALUE SPACES.
           05  RPT-NAME          PIC X(30).
           05  FILLER            PIC X(03) VALUE SPACES.
           05  RPT-AMOUNT        PIC ZZZ,ZZZ,ZZ9.99.

       PROCEDURE DIVISION.
       2000-PROCESS.
           IF WS-LINE-COUNT >= WS-LINES-PER-PAGE
               PERFORM 2800-PAGE-HEADING
           END-IF

           MOVE DEPT-CODE   TO RPT-DEPT
           MOVE EMP-NAME    TO RPT-NAME
           MOVE SALE-AMOUNT TO RPT-AMOUNT

           WRITE REPORT-RECORD FROM RPT-DETAIL-LINE
               AFTER ADVANCING 1 LINE
           ADD 1 TO WS-LINE-COUNT

           PERFORM 9000-READ-INPUT
           .

       2800-PAGE-HEADING.
           ADD 1 TO WS-PAGE-COUNT
           MOVE WS-PAGE-COUNT TO RPT-PAGE
           WRITE REPORT-RECORD FROM RPT-HEADER-1
               AFTER ADVANCING PAGE
           WRITE REPORT-RECORD FROM RPT-HEADER-2
               AFTER ADVANCING 2 LINES
           MOVE 3 TO WS-LINE-COUNT
           .
```

The line counter is initialized to a value at or above the page limit so that the very first record triggers a page heading.

### Checkpoint/Restart

Long-running batch jobs write checkpoints at regular intervals so that a restart after an abend does not reprocess from the beginning. The pattern involves:

1. A **checkpoint counter** that triggers a checkpoint every N records.
2. A **checkpoint file** that stores the current position (key of last processed record) and accumulator values.
3. A **restart routine** that reads the checkpoint file at initialization, repositions input files, and restores accumulators.

```cobol
       WORKING-STORAGE SECTION.
       01  WS-CHECKPOINT-INTERVAL PIC 9(05) VALUE 10000.
       01  WS-RECORDS-SINCE-CHKPT PIC 9(05) VALUE ZEROS.

       01  WS-CHECKPOINT-RECORD.
           05  CHKPT-LAST-KEY    PIC X(10).
           05  CHKPT-REC-COUNT   PIC 9(07).
           05  CHKPT-TOTAL-AMT   PIC S9(11)V99.
           05  CHKPT-TIMESTAMP   PIC X(26).

       PROCEDURE DIVISION.
       1000-INITIALIZE.
           OPEN INPUT  INPUT-FILE
           OPEN OUTPUT OUTPUT-FILE
           OPEN I-O    CHECKPOINT-FILE

      *    Attempt restart
           READ CHECKPOINT-FILE INTO WS-CHECKPOINT-RECORD
               AT END
                   CONTINUE
               NOT AT END
                   PERFORM 1500-RESTART-REPOSITION
           END-READ

           PERFORM 9000-READ-INPUT
           .

       1500-RESTART-REPOSITION.
      *    Skip records already processed
           MOVE CHKPT-REC-COUNT  TO WS-RECORD-COUNT
           MOVE CHKPT-TOTAL-AMT  TO WS-TOTAL-AMOUNT
           PERFORM 9000-READ-INPUT
           PERFORM UNTIL INPUT-KEY >= CHKPT-LAST-KEY
               OR END-OF-FILE
               PERFORM 9000-READ-INPUT
           END-PERFORM
           .

       2000-PROCESS.
      *    -- normal business logic --
           ADD 1 TO WS-RECORDS-SINCE-CHKPT

           IF WS-RECORDS-SINCE-CHKPT >= WS-CHECKPOINT-INTERVAL
               PERFORM 2900-WRITE-CHECKPOINT
               MOVE ZEROS TO WS-RECORDS-SINCE-CHKPT
           END-IF

           PERFORM 9000-READ-INPUT
           .

       2900-WRITE-CHECKPOINT.
           MOVE INPUT-KEY       TO CHKPT-LAST-KEY
           MOVE WS-RECORD-COUNT TO CHKPT-REC-COUNT
           MOVE WS-TOTAL-AMOUNT TO CHKPT-TOTAL-AMT
           MOVE FUNCTION CURRENT-DATE TO CHKPT-TIMESTAMP
           REWRITE CHECKPOINT-RECORD FROM WS-CHECKPOINT-RECORD
           .
```

On mainframes, checkpoint/restart is often coordinated with the operating system or transaction manager (e.g., via `CHKP` macro or DB2 commit points). The application-level pattern shown here works in all environments.

### Multi-File Processing

Programs that consume or produce multiple files extend the sequential pattern. Each file gets its own EOF flag, file status field, and read paragraph.

**Merging two sorted files:**

```cobol
       WORKING-STORAGE SECTION.
       01  WS-FILE-A-EOF        PIC X VALUE 'N'.
           88 FILE-A-EOF                VALUE 'Y'.
       01  WS-FILE-B-EOF        PIC X VALUE 'N'.
           88 FILE-B-EOF                VALUE 'Y'.

       PROCEDURE DIVISION.
       0000-MAIN.
           PERFORM 1000-INITIALIZE
           PERFORM 2000-MERGE-PROCESS
               UNTIL FILE-A-EOF AND FILE-B-EOF
           PERFORM 3000-TERMINATE
           STOP RUN
           .

       2000-MERGE-PROCESS.
           EVALUATE TRUE
               WHEN KEY-A < KEY-B
                   WRITE OUTPUT-RECORD FROM RECORD-A
                   PERFORM 9100-READ-FILE-A

               WHEN KEY-A > KEY-B
                   WRITE OUTPUT-RECORD FROM RECORD-B
                   PERFORM 9200-READ-FILE-B

               WHEN KEY-A = KEY-B
      *            Duplicate key -- take A, skip B (or merge)
                   WRITE OUTPUT-RECORD FROM RECORD-A
                   PERFORM 9100-READ-FILE-A
                   PERFORM 9200-READ-FILE-B
           END-EVALUATE
           .
```

The same `HIGH-VALUES` drain technique from the master-transaction pattern applies here.

**Splitting a single input into multiple outputs** (e.g., routing records by type):

```cobol
       2000-PROCESS.
           EVALUATE RECORD-TYPE OF INPUT-RECORD
               WHEN 'A'
                   WRITE TYPE-A-RECORD FROM INPUT-RECORD
               WHEN 'B'
                   WRITE TYPE-B-RECORD FROM INPUT-RECORD
               WHEN OTHER
                   WRITE REJECT-RECORD FROM INPUT-RECORD
                   ADD 1 TO WS-REJECT-COUNT
           END-EVALUATE
           ADD 1 TO WS-RECORD-COUNT
           PERFORM 9000-READ-INPUT
           .
```

### Header/Footer/Detail Record Processing

Many legacy file formats embed header, detail, and footer (trailer) records in the same file, distinguished by a record-type code in a fixed position.

```cobol
       FD  INPUT-FILE
           RECORDING MODE IS F.
       01  INPUT-RECORD.
           05  IN-REC-TYPE       PIC X.
               88 IS-HEADER              VALUE 'H'.
               88 IS-DETAIL              VALUE 'D'.
               88 IS-TRAILER             VALUE 'T'.
           05  IN-REC-BODY       PIC X(79).

       WORKING-STORAGE SECTION.
       01  WS-HEADER-DATA.
           05  HDR-RUN-DATE      PIC X(08).
           05  HDR-BATCH-ID      PIC X(10).

       01  WS-TRAILER-DATA.
           05  TRL-RECORD-COUNT  PIC 9(07).
           05  TRL-HASH-TOTAL    PIC 9(11).

       01  WS-COMPUTED-COUNT     PIC 9(07) VALUE ZEROS.

       PROCEDURE DIVISION.
       2000-PROCESS.
           EVALUATE TRUE
               WHEN IS-HEADER
                   PERFORM 2100-PROCESS-HEADER
               WHEN IS-DETAIL
                   PERFORM 2200-PROCESS-DETAIL
               WHEN IS-TRAILER
                   PERFORM 2300-PROCESS-TRAILER
               WHEN OTHER
                   PERFORM 2900-INVALID-RECORD
           END-EVALUATE
           PERFORM 9000-READ-INPUT
           .

       2100-PROCESS-HEADER.
           MOVE IN-REC-BODY(1:08) TO HDR-RUN-DATE
           MOVE IN-REC-BODY(9:10) TO HDR-BATCH-ID
           .

       2200-PROCESS-DETAIL.
           ADD 1 TO WS-COMPUTED-COUNT
      *    -- process the detail record --
           .

       2300-PROCESS-TRAILER.
           MOVE IN-REC-BODY(1:07) TO TRL-RECORD-COUNT
           IF WS-COMPUTED-COUNT NOT = TRL-RECORD-COUNT
               DISPLAY 'RECORD COUNT MISMATCH: EXPECTED '
                   TRL-RECORD-COUNT ' GOT ' WS-COMPUTED-COUNT
               MOVE 12 TO RETURN-CODE
           END-IF
           .
```

The trailer record validation is a critical integrity check. It ensures no records were lost or duplicated during file transfer or prior processing.

## Common Patterns

- **Priming read** -- Always read the first record before entering the main loop. This correctly handles empty files and avoids processing stale data in the record buffer.

- **Trailing read** -- Place the next `READ` at the end of the processing paragraph, not at the beginning. This ensures the AT END flag is set before the loop condition is re-evaluated.

- **HIGH-VALUES sentinel** -- When a file reaches EOF, set its key to `HIGH-VALUES` so the comparison logic naturally drains the other file. This eliminates the need for separate "drain" loops.

- **Return code setting** -- Batch programs communicate success or failure to JCL through the `RETURN-CODE` special register. See [error_handling.md](error_handling.md) for the complete RETURN-CODE reference and conventional values.

- **Sort-as-a-tool** -- When input is not pre-sorted, use the COBOL `SORT` verb with `INPUT PROCEDURE` and `OUTPUT PROCEDURE` to sort inline. This avoids a separate sort step in JCL.

- **Balanced line algorithm** -- The master-transaction match pattern is sometimes called the "balanced line" algorithm. It ensures every master and every transaction is processed exactly once.

- **Work file separation** -- When a program needs to make multiple passes over data, write an intermediate work file in the first pass and read it back in the second. This is more reliable than attempting to rewind sequential files.

- **Paragraph numbering** -- Batch programs commonly use numeric paragraph names (1000, 2000, 2100, etc.) to indicate hierarchy and call order. This is a convention, not a language requirement, but it aids readability enormously in programs with dozens of paragraphs.

- **INITIALIZE vs MOVE ZEROS** -- Use `INITIALIZE` on group items to reset all elementary fields to their category defaults (spaces for alphanumeric, zeros for numeric). Use `MOVE ZEROS` only for individual numeric fields. Be aware that `INITIALIZE` does not reset `FILLER` items in some compilers.

- **Record counting for every file** -- Maintain a separate record count for every input and output file. Log all counts at termination. This is the first line of defense when a batch run produces unexpected results.

- **Error file / reject file** -- Write records that fail validation to a separate reject file rather than terminating the program. Process the reject file in a follow-up step or manual review. Always include the reason code with the rejected record.

## Gotchas

- **Missing priming read causes processing of garbage.** If you enter the main loop without a priming read, the record area contains whatever was in memory (often LOW-VALUES or residual data from a prior program in the same run unit). The program will process one garbage record before reading real data.

- **Forgetting the final control break.** When the last group of records is processed, no key change triggers the break. The termination paragraph must explicitly force the final break at every level. Missing this causes the last group's subtotals to be lost from the report.

- **Not setting HIGH-VALUES at EOF in matching programs.** If the EOF key is left as the last record's key rather than HIGH-VALUES, the comparison logic may skip records from the other file or loop infinitely, depending on the key relationship.

- **Testing EOF flag before the read sets it.** Placing the read at the top of the process paragraph and testing the flag at the bottom (or vice versa) can cause off-by-one processing --- either the last record is skipped or an extra iteration processes stale data.

- **Accumulator overflow.** If an accumulator field is too small for the volume of data, it will silently truncate on overflow (SIZE ERROR is not raised unless explicitly coded). Always size accumulators generously and consider using `ON SIZE ERROR` for critical totals.

```cobol
           ADD AMOUNT TO WS-GRAND-TOTAL
               ON SIZE ERROR
                   DISPLAY 'ACCUMULATOR OVERFLOW ON GRAND TOTAL'
                   MOVE 16 TO RETURN-CODE
                   PERFORM 3000-TERMINATE
                   STOP RUN
           END-ADD
```

- **File status not checked after OPEN/CLOSE.** A failed OPEN (status 35 = file not found, status 37 = file attribute conflict) will not cause an abend on all systems. The program may continue with an unopened file, producing empty output or misleading results. Always check file status after every I/O operation.

- **Control break on the wrong field.** If the input is sorted by REGION/BRANCH/DEPT but the break logic checks DEPT before REGION, a DEPT value that repeats across regions will not trigger a break when it should.

- **Multiple transactions for the same master.** In the master-transaction match, after applying a transaction, read the next transaction --- but do NOT read the next master. The next transaction might also match the same master. Only advance the master when the transaction key is greater.

- **Checkpoint file not cleared on successful completion.** If the checkpoint file still contains data from a successful run, the next execution will incorrectly attempt a restart. Always clear or delete the checkpoint file at program termination after a successful run.

- **REWRITE without prior READ.** The `REWRITE` verb requires that the record was read by the most recent `READ` on that file. Attempting to `REWRITE` without a prior `READ` of the same record causes a runtime error.

- **Assuming SPACES in numeric comparisons.** When testing whether a previous-key field has been set, comparing a numeric field to SPACES is invalid. Use a separate "first-time" flag or initialize the previous-key field to a known impossible value.

- **WRITE AFTER ADVANCING on a closed file.** If the report file is closed before the final totals are written, the WRITE silently fails on some systems and abends on others. Sequence the termination logic carefully: totals first, then CLOSE.

## Related Topics

- **file_handling.md** -- Detailed coverage of OPEN, CLOSE, READ, WRITE, REWRITE, DELETE verbs and file status codes. Batch patterns build directly on these I/O operations.
- **jcl_interaction.md** -- How batch programs receive file assignments (DD statements), condition code testing between job steps, and SORT/MERGE integration with JCL.
- **io_processing.md** -- Low-level details of record buffering, blocking factors, RECORDING MODE, and ORGANIZATION clauses that affect batch throughput.
- **paragraph_flow.md** -- PERFORM mechanics (PERFORM UNTIL, inline PERFORM, nested PERFORM) that drive the control flow of every batch pattern documented here.
- **error_handling.md** -- Comprehensive coverage of FILE STATUS checking, USE AFTER EXCEPTION declaratives, and abend handling strategies that complement the error patterns in batch processing.
