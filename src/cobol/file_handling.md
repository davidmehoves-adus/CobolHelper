# File Handling

## Description
COBOL file handling encompasses the statements, clauses, and structures used to define, open, read, write, update, and close data files. It is one of the most critical aspects of COBOL programming, as the vast majority of batch COBOL programs exist to process files. This reference covers file organization types, SELECT/ASSIGN clauses, FD entries, all file I/O verbs (OPEN, CLOSE, READ, WRITE, REWRITE, DELETE, START), file status codes, and related clauses such as BLOCK CONTAINS and RECORD CONTAINS.

## Table of Contents
- [Core Concepts](#core-concepts)
  - [File Organization Types](#file-organization-types)
  - [Access Modes](#access-modes)
  - [SELECT and ASSIGN](#select-and-assign)
  - [FD Entries](#fd-entries)
  - [BLOCK CONTAINS](#block-contains)
  - [RECORD CONTAINS](#record-contains)
  - [File Status](#file-status)
- [Syntax & Examples](#syntax--examples)
  - [OPEN Statement](#open-statement)
  - [CLOSE Statement](#close-statement)
  - [READ Statement](#read-statement)
  - [WRITE Statement](#write-statement)
  - [REWRITE Statement](#rewrite-statement)
  - [DELETE Statement](#delete-statement)
  - [START Statement](#start-statement)
  - [Full File Processing Example](#full-file-processing-example)
- [Common Patterns](#common-patterns)
- [Gotchas](#gotchas)
- [Related Topics](#related-topics)

## Core Concepts

### File Organization Types

COBOL supports three fundamental file organizations, each determining how records are physically stored and logically accessed.

**SEQUENTIAL** organization stores records one after another in the order they were written. Records can only be accessed in sequence. This is the simplest and most common organization, used for flat files, reports, and tape-based data. Sequential files support only sequential access mode.

**INDEXED** organization stores records with one or more key fields that allow direct retrieval. Each indexed file has a primary key (which must be unique) and may have one or more alternate keys (which can optionally allow duplicates). Indexed files support sequential, random, and dynamic access modes. On mainframes, indexed files are typically implemented as VSAM KSDS (Key-Sequenced Data Sets).

**RELATIVE** organization stores records in numbered slots identified by a relative record number (an integer starting at 1). Records can be accessed by their relative position. Relative files support sequential, random, and dynamic access modes. On mainframes, relative files are typically implemented as VSAM RRDS (Relative Record Data Sets).

### Access Modes

The ACCESS MODE clause in the SELECT statement determines how a program reads or writes records.

- **SEQUENTIAL** -- Records are accessed in order (physical order for sequential files, key order for indexed files, relative record number order for relative files). This is the default if ACCESS MODE is not specified.
- **RANDOM** -- Records are accessed by key value (indexed) or relative record number (relative). Each I/O operation specifies which record to access.
- **DYNAMIC** -- Allows both sequential and random access within the same program. The program can switch between sequential reads (READ NEXT) and random reads (READ with key) as needed. Only available for indexed and relative files.

### SELECT and ASSIGN

The SELECT clause in the ENVIRONMENT DIVISION's FILE-CONTROL paragraph associates a logical file name used in the program with an external file. The ASSIGN clause links the logical file to a physical file or device.

```cobol
       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT file-name
               ASSIGN TO assignment-name
               ORGANIZATION IS org-type
               ACCESS MODE IS access-type
               RECORD KEY IS key-field
               ALTERNATE RECORD KEY IS alt-key
                   WITH DUPLICATES
               FILE STATUS IS ws-file-status.
```

Key clauses within SELECT:

- **ASSIGN TO** -- Specifies the external file name or DD name (on mainframes, this maps to a JCL DD statement). The exact syntax varies by platform, but the identifier typically references a JCL DD name on mainframes or a file path on other platforms.
- **ORGANIZATION IS** -- Specifies SEQUENTIAL, INDEXED, or RELATIVE. Defaults to SEQUENTIAL if omitted.
- **ACCESS MODE IS** -- Specifies SEQUENTIAL, RANDOM, or DYNAMIC. Defaults to SEQUENTIAL if omitted.
- **RECORD KEY IS** -- Required for INDEXED organization. Identifies the primary key field defined in the FD record description. The field must be within the record area.
- **ALTERNATE RECORD KEY IS** -- Optional for INDEXED files. Defines alternate keys for secondary access paths. The WITH DUPLICATES phrase allows non-unique values in the alternate key.
- **RELATIVE KEY IS** -- Used with RELATIVE organization when access mode is RANDOM or DYNAMIC. Identifies a numeric field (not part of the record) that holds the relative record number.
- **FILE STATUS IS** -- Identifies a two-byte alphanumeric field in WORKING-STORAGE that receives a status code after each I/O operation. Strongly recommended for all files.

### FD Entries

The FD (File Description) entry in the DATA DIVISION's FILE SECTION describes the logical structure of a file. It connects the SELECT clause to the record layout.

```cobol
       DATA DIVISION.
       FILE SECTION.
       FD  CUSTOMER-FILE
           RECORDING MODE IS F
           BLOCK CONTAINS 0 RECORDS
           RECORD CONTAINS 200 CHARACTERS
           LABEL RECORDS ARE STANDARD
           DATA RECORD IS CUSTOMER-RECORD.
       01  CUSTOMER-RECORD.
           05  CUST-ID            PIC X(10).
           05  CUST-NAME          PIC X(50).
           05  CUST-ADDRESS       PIC X(100).
           05  CUST-BALANCE       PIC S9(9)V99 COMP-3.
           05  FILLER             PIC X(34).
```

Key FD clauses:

- **RECORDING MODE** -- Specifies record format: F (fixed), V (variable), U (undefined), or S (spanned). Primarily relevant on mainframes.
- **BLOCK CONTAINS** -- See dedicated section below.
- **RECORD CONTAINS** -- See dedicated section below.
- **LABEL RECORDS** -- Historically specified whether the file has standard labels. In modern COBOL, STANDARD is assumed and this clause is treated as commentary.
- **DATA RECORD IS** -- Documents which 01-level record(s) belong to this FD. Treated as commentary in modern COBOL.

An FD can have multiple 01-level record descriptions, which implicitly redefine each other (they share the same record buffer).

### BLOCK CONTAINS

The BLOCK CONTAINS clause specifies how many records (or characters) are grouped into a physical block on the storage medium. Blocking improves I/O performance by reducing the number of physical read/write operations.

```cobol
       FD  TRANS-FILE
           BLOCK CONTAINS 0 RECORDS.
```

- **BLOCK CONTAINS 0 RECORDS** -- Tells the system to determine the optimal block size automatically. On mainframes, this defers to the JCL or catalog. This is the recommended practice in almost all cases.
- **BLOCK CONTAINS n RECORDS** -- Specifies exactly n logical records per physical block.
- **BLOCK CONTAINS n CHARACTERS** -- Specifies block size in characters (bytes).

When BLOCK CONTAINS is omitted, the file is treated as unblocked (one record per block), which is extremely inefficient for sequential files on mainframes.

### RECORD CONTAINS

The RECORD CONTAINS clause specifies the record length.

```cobol
      * Fixed-length records
       FD  FIXED-FILE
           RECORD CONTAINS 80 CHARACTERS.

      * Variable-length records
       FD  VARIABLE-FILE
           RECORD IS VARYING IN SIZE
               FROM 50 TO 500 CHARACTERS
               DEPENDING ON WS-REC-LEN.
```

- **RECORD CONTAINS n CHARACTERS** -- Fixed-length records of exactly n bytes.
- **RECORD IS VARYING IN SIZE FROM n TO m CHARACTERS DEPENDING ON identifier** -- Variable-length records. The DEPENDING ON identifier is a numeric field that the system sets to the actual record length on READ and that the program sets before WRITE.
- If RECORD CONTAINS is omitted, the record length is implied from the 01-level record descriptions under the FD.

### File Status

The FILE STATUS clause designates a two-character alphanumeric field that receives a return code after every I/O operation on the file. Checking file status after each I/O operation is considered essential for robust COBOL programs.

The status code is composed of two characters: status key 1 (the first character, indicating the general category) and status key 2 (the second character, providing specifics).

**Status Key 1 Categories:**

| Key 1 | Meaning |
|-------|---------|
| 0     | Successful completion |
| 1     | AT END condition |
| 2     | Invalid key condition |
| 3     | Permanent error |
| 4     | Logic error |
| 9     | Implementor-defined |

**Comprehensive File Status Codes:**

| Code | Meaning |
|------|---------|
| 00   | Successful completion. The I/O operation completed without error. |
| 02   | Successful completion, duplicate key detected. A READ or WRITE on an indexed file succeeded, but a duplicate exists for an alternate key that allows duplicates. |
| 04   | Successful READ, but the record length does not match the FD fixed-length specification. The record was read but may be truncated or padded. |
| 05   | Successful OPEN, but the optional file was not present. The file has been created (for OUTPUT/I-O/EXTEND) or treated as empty (for INPUT). |
| 07   | Successful operation on a file opened with NO REWIND or for a tape device. Implementation-specific. |
| 10   | End of file (AT END). A sequential READ attempted to read beyond the last record. This is a normal condition in sequential processing. |
| 14   | Relative file sequential READ: the relative record number exceeds the maximum allowed for the file. |
| 21   | Sequence error. For an indexed file opened in sequential access mode, the record key is not in ascending order during a WRITE, or the record key was changed before a REWRITE. |
| 22   | Duplicate key. An attempt to WRITE or REWRITE a record with a duplicate primary key, or a duplicate alternate key where duplicates are not allowed. |
| 23   | Record not found. A READ, START, or DELETE attempted to access a record by key, but no matching record exists. |
| 24   | Boundary violation. An attempt to WRITE beyond the file boundaries (e.g., disk full, or writing past the maximum relative record number). |
| 30   | Permanent I/O error. A non-recoverable error occurred; no further detail is available. Typically a hardware or system-level failure. |
| 34   | Boundary violation on a sequential WRITE. The file has exceeded its allocated space (disk full or allocation exceeded). |
| 35   | OPEN failed because the file does not exist and the OPEN mode requires it (INPUT or I-O on a non-optional file). |
| 37   | OPEN failed due to a conflict between the file attributes (organization, access mode, key definitions) and the actual file on disk. |
| 38   | OPEN failed because the file was previously closed WITH LOCK. |
| 39   | OPEN failed due to a mismatch between file attributes specified in the program and the actual file attributes (e.g., record length, key length, key position). |
| 41   | OPEN failed because the file is already open. |
| 42   | CLOSE failed because the file is not open. |
| 43   | In sequential access mode, the last I/O operation was not a successful READ before a REWRITE or DELETE was attempted. |
| 44   | Boundary violation. A WRITE or REWRITE attempted to write a record whose length is outside the allowed range for the file. |
| 46   | A sequential READ was attempted but the preceding operation (READ or START) was unsuccessful, so the current file position is undefined. |
| 47   | A READ or START was attempted on a file not opened in INPUT or I-O mode. |
| 48   | A WRITE was attempted on a file not opened in OUTPUT, I-O, or EXTEND mode. |
| 49   | A REWRITE or DELETE was attempted on a file not opened in I-O mode. |
| 90   | Implementation-defined. Typically used by specific compilers for additional error detail. |
| 91   | Implementation-defined. On some systems, indicates a VSAM password failure or similar security issue. |
| 92   | Implementation-defined. Logic error, often related to conflicting file operations. |
| 93   | Implementation-defined. Resource unavailable (e.g., VSAM resource conflict). |
| 94   | Implementation-defined. Current record pointer undefined or sequential read not available. |
| 95   | Implementation-defined. Incomplete or invalid file information. |
| 96   | Implementation-defined. No DD statement found (mainframe) or file assignment missing. |
| 97   | Implementation-defined. Successful OPEN but file integrity has been verified. |

Note: Status codes 90-99 are implementation-defined and vary between compilers. Always consult your specific compiler documentation for these codes.

## Syntax & Examples

### OPEN Statement

The OPEN statement makes a file available for processing. A file must be opened before any other I/O operations can be performed on it.

```cobol
       OPEN INPUT  input-file
       OPEN OUTPUT output-file
       OPEN I-O    update-file
       OPEN EXTEND append-file
```

**OPEN modes:**

- **INPUT** -- Opens the file for reading only. The file must already exist (unless declared as OPTIONAL in the SELECT clause). The file pointer is positioned at the beginning.
- **OUTPUT** -- Opens the file for writing only. If the file exists, its contents are erased. If it does not exist, it is created. Records can only be written.
- **I-O** -- Opens the file for both reading and writing. Allows READ, WRITE (indexed/relative only), REWRITE, and DELETE. The file must already exist. Used for in-place updates.
- **EXTEND** -- Opens the file for appending. The file pointer is positioned after the last existing record. New records are written after existing content. Equivalent to opening for OUTPUT but preserving existing data.

Multiple files can be opened in a single OPEN statement:

```cobol
       OPEN INPUT  CUSTOMER-FILE
                   TRANSACTION-FILE
            OUTPUT REPORT-FILE
            I-O    MASTER-FILE
```

### CLOSE Statement

The CLOSE statement terminates processing on an open file, flushes buffers, and releases system resources.

```cobol
       CLOSE file-name-1
       CLOSE file-name-1 file-name-2 file-name-3
       CLOSE file-name-1 WITH LOCK
```

- **CLOSE file-name** -- Standard close. Flushes all buffers and releases the file.
- **CLOSE WITH LOCK** -- Closes the file and prevents it from being reopened in the same program execution. Useful for ensuring a file is processed only once.
- Multiple files can be closed in a single statement.
- Closing a file that is not open results in file status 42.

### READ Statement

The READ statement retrieves records from an open file.

**Sequential READ:**

```cobol
       READ input-file INTO ws-record
           AT END
               SET end-of-file TO TRUE
           NOT AT END
               PERFORM process-record
       END-READ
```

**Sequential READ NEXT (for DYNAMIC access mode):**

```cobol
       READ indexed-file NEXT RECORD INTO ws-record
           AT END
               SET end-of-file TO TRUE
       END-READ
```

**Random READ (indexed file by primary key):**

```cobol
       MOVE '12345' TO CUST-ID
       READ customer-file INTO ws-customer
           KEY IS CUST-ID
           INVALID KEY
               DISPLAY 'Customer not found'
           NOT INVALID KEY
               PERFORM process-customer
       END-READ
```

**Random READ (indexed file by alternate key):**

```cobol
       MOVE 'SMITH' TO CUST-LAST-NAME
       READ customer-file INTO ws-customer
           KEY IS CUST-LAST-NAME
           INVALID KEY
               DISPLAY 'No customer with that name'
       END-READ
```

**Random READ (relative file):**

```cobol
       MOVE 42 TO WS-RELATIVE-KEY
       READ relative-file INTO ws-record
           INVALID KEY
               DISPLAY 'Slot 42 is empty'
       END-READ
```

Key points:
- The INTO clause copies the record from the file buffer into a working-storage field. Without INTO, data is read directly into the FD record area.
- AT END triggers when reading past the last record in sequential mode.
- INVALID KEY triggers when a random READ cannot find a matching record.
- After a successful READ, the record is available in both the FD record area and (if INTO was used) the working-storage field.

### WRITE Statement

The WRITE statement outputs a record to a file. Note that WRITE operates on a record name (the 01-level under the FD), not the file name.

**Sequential WRITE:**

```cobol
       WRITE output-record FROM ws-detail-line
           INVALID KEY
               DISPLAY 'Write error'
       END-WRITE
```

**WRITE with ADVANCING (for report files):**

```cobol
       WRITE print-line FROM ws-header
           AFTER ADVANCING PAGE
       WRITE print-line FROM ws-detail
           AFTER ADVANCING 1 LINE
       WRITE print-line FROM ws-detail
           BEFORE ADVANCING 2 LINES
```

**Random WRITE (indexed file with I-O or OUTPUT):**

```cobol
       MOVE ws-new-customer TO customer-record
       WRITE customer-record
           INVALID KEY
               DISPLAY 'Duplicate key: ' CUST-ID
       END-WRITE
```

Key points:
- WRITE uses the record name, not the file name (unlike READ, which uses the file name).
- The FROM clause moves data from working-storage to the record area before writing.
- ADVANCING controls line spacing for printer files. AFTER ADVANCING PAGE triggers a page eject.
- INVALID KEY applies to indexed and relative files and triggers on duplicate keys or boundary violations.
- For sequential files opened OUTPUT, records are written in the order the WRITE statements execute.

### REWRITE Statement

The REWRITE statement updates an existing record in place. The file must be opened I-O.

```cobol
       READ master-file INTO ws-master
           INVALID KEY
               DISPLAY 'Record not found'
       END-READ

       MOVE new-balance TO ws-master-balance
       MOVE ws-master TO master-record
       REWRITE master-record FROM ws-master
           INVALID KEY
               DISPLAY 'Rewrite failed'
       END-REWRITE
```

Key points:
- For sequential access mode, a successful READ must immediately precede the REWRITE (status 43 results otherwise).
- For indexed files in sequential access mode, the primary key value must not be changed before the REWRITE.
- For indexed files in random or dynamic access mode, the record to be rewritten is identified by the primary key value in the record, and no prior READ is required.
- The record length on REWRITE must match the original for fixed-length records.

### DELETE Statement

The DELETE statement removes a record from an indexed or relative file. The file must be opened I-O. DELETE is not valid for sequential files.

```cobol
      * Sequential access: DELETE the record just read
       READ customer-file
           INVALID KEY
               DISPLAY 'Not found'
       END-READ
       DELETE customer-file RECORD
           INVALID KEY
               DISPLAY 'Delete failed'
       END-DELETE

      * Random access: DELETE by key
       MOVE '12345' TO CUST-ID
       DELETE customer-file RECORD
           INVALID KEY
               DISPLAY 'Record not found for delete'
       END-DELETE
```

Key points:
- In sequential access mode, the record to be deleted is the one most recently read; a successful READ must precede the DELETE.
- In random or dynamic access mode, the record is identified by the current value of the primary key.
- DELETE operates on the file name, not the record name.

### START Statement

The START statement positions the file pointer within an indexed or relative file for subsequent sequential reads. It does not read a record; it only establishes position.

```cobol
       MOVE 'M' TO CUST-ID
       START customer-file
           KEY IS GREATER THAN OR EQUAL TO CUST-ID
           INVALID KEY
               DISPLAY 'No records from M onward'
       END-START

       PERFORM UNTIL end-of-file
           READ customer-file NEXT INTO ws-customer
               AT END
                   SET end-of-file TO TRUE
               NOT AT END
                   PERFORM process-customer
           END-READ
       END-PERFORM
```

Key relational operators for START:
- `KEY IS EQUAL TO`
- `KEY IS GREATER THAN`
- `KEY IS NOT LESS THAN` (same as GREATER THAN OR EQUAL TO)
- `KEY IS GREATER THAN OR EQUAL TO`

Key points:
- START is only valid for indexed and relative files.
- The file must be opened INPUT or I-O.
- START does not retrieve a record. Use READ NEXT after START.
- START positions to the first record satisfying the key condition.
- INVALID KEY triggers if no record satisfies the condition.

### Full File Processing Example

A complete program structure showing sequential file processing with proper file status checking:

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. FILE-PROCESS.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT INPUT-FILE
               ASSIGN TO INFILE
               ORGANIZATION IS SEQUENTIAL
               FILE STATUS IS WS-INPUT-STATUS.
           SELECT OUTPUT-FILE
               ASSIGN TO OUTFILE
               ORGANIZATION IS SEQUENTIAL
               FILE STATUS IS WS-OUTPUT-STATUS.
           SELECT MASTER-FILE
               ASSIGN TO MASTER
               ORGANIZATION IS INDEXED
               ACCESS MODE IS DYNAMIC
               RECORD KEY IS MASTER-KEY
               ALTERNATE RECORD KEY IS MASTER-NAME
                   WITH DUPLICATES
               FILE STATUS IS WS-MASTER-STATUS.

       DATA DIVISION.
       FILE SECTION.
       FD  INPUT-FILE
           BLOCK CONTAINS 0 RECORDS
           RECORD CONTAINS 80 CHARACTERS.
       01  INPUT-RECORD              PIC X(80).

       FD  OUTPUT-FILE
           BLOCK CONTAINS 0 RECORDS
           RECORD CONTAINS 132 CHARACTERS.
       01  OUTPUT-RECORD             PIC X(132).

       FD  MASTER-FILE
           RECORD CONTAINS 200 CHARACTERS.
       01  MASTER-RECORD.
           05  MASTER-KEY            PIC X(10).
           05  MASTER-NAME           PIC X(40).
           05  MASTER-DATA           PIC X(150).

       WORKING-STORAGE SECTION.
       01  WS-INPUT-STATUS           PIC X(02).
           88  INPUT-OK              VALUE '00'.
           88  INPUT-EOF             VALUE '10'.
       01  WS-OUTPUT-STATUS          PIC X(02).
           88  OUTPUT-OK             VALUE '00'.
       01  WS-MASTER-STATUS          PIC X(02).
           88  MASTER-OK             VALUE '00'.
           88  MASTER-NOT-FOUND      VALUE '23'.
       01  WS-INPUT-REC              PIC X(80).

       PROCEDURE DIVISION.
       0000-MAIN.
           PERFORM 1000-INITIALIZE
           PERFORM 2000-PROCESS
               UNTIL INPUT-EOF
           PERFORM 9000-TERMINATE
           STOP RUN
           .

       1000-INITIALIZE.
           OPEN INPUT  INPUT-FILE
           IF NOT INPUT-OK
               DISPLAY 'Error opening INPUT-FILE: '
                       WS-INPUT-STATUS
               STOP RUN
           END-IF

           OPEN OUTPUT OUTPUT-FILE
           IF NOT OUTPUT-OK
               DISPLAY 'Error opening OUTPUT-FILE: '
                       WS-OUTPUT-STATUS
               STOP RUN
           END-IF

           OPEN I-O MASTER-FILE
           IF NOT MASTER-OK
               DISPLAY 'Error opening MASTER-FILE: '
                       WS-MASTER-STATUS
               STOP RUN
           END-IF

           PERFORM 8000-READ-INPUT
           .

       2000-PROCESS.
           PERFORM 2100-LOOKUP-MASTER
           PERFORM 2200-WRITE-OUTPUT
           PERFORM 8000-READ-INPUT
           .

       2100-LOOKUP-MASTER.
           MOVE WS-INPUT-REC(1:10) TO MASTER-KEY
           READ MASTER-FILE
               INVALID KEY
                   CONTINUE
           END-READ
           IF MASTER-NOT-FOUND
               DISPLAY 'Master record not found: '
                       MASTER-KEY
           END-IF
           .

       2200-WRITE-OUTPUT.
           MOVE SPACES TO OUTPUT-RECORD
           MOVE WS-INPUT-REC TO OUTPUT-RECORD(1:80)
           WRITE OUTPUT-RECORD
           IF NOT OUTPUT-OK
               DISPLAY 'Write error: ' WS-OUTPUT-STATUS
           END-IF
           .

       8000-READ-INPUT.
           READ INPUT-FILE INTO WS-INPUT-REC
               AT END
                   CONTINUE
           END-READ
           IF NOT INPUT-OK AND NOT INPUT-EOF
               DISPLAY 'Read error: ' WS-INPUT-STATUS
               STOP RUN
           END-IF
           .

       9000-TERMINATE.
           CLOSE INPUT-FILE
                 OUTPUT-FILE
                 MASTER-FILE
           .
```

## Common Patterns

### Sequential File Copy with Transformation

The most basic pattern: read every record from an input file, transform it, and write to an output file. This is the backbone of batch processing.

```cobol
       PERFORM UNTIL WS-EOF
           READ INPUT-FILE INTO WS-INPUT-REC
               AT END
                   SET WS-EOF TO TRUE
           END-READ
           IF NOT WS-EOF
               PERFORM TRANSFORM-RECORD
               WRITE OUTPUT-RECORD FROM WS-OUTPUT-REC
           END-IF
       END-PERFORM
```

### Match-Merge Processing

A classic pattern where two sorted sequential files are processed together, matching records on a common key. This is one of the most fundamental batch COBOL patterns.

```cobol
       PERFORM UNTIL BOTH-FILES-DONE
           EVALUATE TRUE
               WHEN MASTER-KEY < TRANS-KEY
                   PERFORM WRITE-UNMATCHED-MASTER
                   PERFORM READ-MASTER
               WHEN MASTER-KEY = TRANS-KEY
                   PERFORM PROCESS-MATCHED-RECORDS
                   PERFORM READ-MASTER
                   PERFORM READ-TRANSACTION
               WHEN MASTER-KEY > TRANS-KEY
                   PERFORM WRITE-UNMATCHED-TRANS
                   PERFORM READ-TRANSACTION
           END-EVALUATE
       END-PERFORM
```

### Indexed File Random Update

Read transactions and apply updates to a master file using random access.

```cobol
       PERFORM UNTIL NO-MORE-TRANS
           READ TRANS-FILE INTO WS-TRANS
               AT END
                   SET NO-MORE-TRANS TO TRUE
           END-READ
           IF NOT NO-MORE-TRANS
               MOVE TRANS-KEY TO MASTER-KEY
               READ MASTER-FILE
                   INVALID KEY
                       PERFORM ADD-NEW-MASTER
                   NOT INVALID KEY
                       PERFORM UPDATE-EXISTING-MASTER
               END-READ
           END-IF
       END-PERFORM
```

### File Status Checking Paragraph

A reusable pattern for centralized file status checking, commonly used in production programs to standardize error handling.

```cobol
       9500-CHECK-STATUS.
           EVALUATE WS-FILE-STATUS
               WHEN '00'
                   CONTINUE
               WHEN '10'
                   SET WS-END-OF-FILE TO TRUE
               WHEN '23'
                   SET WS-REC-NOT-FOUND TO TRUE
               WHEN OTHER
                   DISPLAY 'FILE I/O ERROR'
                   DISPLAY 'FILE: ' WS-FILE-NAME
                   DISPLAY 'STATUS: ' WS-FILE-STATUS
                   DISPLAY 'OPERATION: ' WS-IO-OPERATION
                   MOVE 16 TO RETURN-CODE
                   STOP RUN
           END-EVALUATE
           .
```

### EXTEND Mode for Appending to Logs

Production programs often append records to cumulative log or audit files using EXTEND mode.

```cobol
       OPEN EXTEND AUDIT-LOG-FILE
       IF WS-AUDIT-STATUS NOT = '00'
           IF WS-AUDIT-STATUS = '35'
               OPEN OUTPUT AUDIT-LOG-FILE
           ELSE
               DISPLAY 'Cannot open audit log: '
                       WS-AUDIT-STATUS
               STOP RUN
           END-IF
       END-IF

       MOVE FUNCTION CURRENT-DATE TO WS-TIMESTAMP
       STRING WS-TIMESTAMP DELIMITED SIZE
              ' ' DELIMITED SIZE
              WS-AUDIT-MESSAGE DELIMITED SPACE
              INTO AUDIT-RECORD
       END-STRING
       WRITE AUDIT-RECORD
       CLOSE AUDIT-LOG-FILE
```

### Browse by Alternate Key with START

Position into an indexed file by alternate key and read forward sequentially -- a pattern often used for inquiry-type processing.

```cobol
       MOVE WS-SEARCH-NAME TO CUST-NAME-ALT-KEY
       START CUSTOMER-FILE
           KEY IS NOT LESS THAN CUST-NAME-ALT-KEY
           INVALID KEY
               DISPLAY 'No matching customers'
               SET WS-BROWSE-DONE TO TRUE
       END-START

       PERFORM UNTIL WS-BROWSE-DONE
                  OR WS-RECORDS-DISPLAYED >= 20
           READ CUSTOMER-FILE NEXT INTO WS-CUST
               AT END
                   SET WS-BROWSE-DONE TO TRUE
           END-READ
           IF NOT WS-BROWSE-DONE
               ADD 1 TO WS-RECORDS-DISPLAYED
               PERFORM DISPLAY-CUSTOMER
           END-IF
       END-PERFORM
```

## Gotchas

- **Forgetting to check file status after every I/O operation.** Without file status checking, a program may continue processing after an I/O error, silently producing corrupt output. Many production abends trace back to an unchecked file status. Always define FILE STATUS IS and check it after every OPEN, READ, WRITE, REWRITE, DELETE, START, and CLOSE.

- **WRITE uses the record name, READ uses the file name.** This inconsistency is one of the most common sources of compiler errors for COBOL beginners. `READ CUSTOMER-FILE` is correct. `WRITE CUSTOMER-RECORD` is correct. Reversing them produces a compile error.

- **Omitting BLOCK CONTAINS 0 RECORDS on sequential files.** Without this clause, many compilers default to unblocked I/O (one record per physical block), which can degrade performance by 10x or more on mainframes. Always specify `BLOCK CONTAINS 0 RECORDS` to allow the system to choose the optimal block size.

- **Opening a file OUTPUT when you meant EXTEND.** OPEN OUTPUT destroys existing file contents. If the intent is to append records to an existing file, use OPEN EXTEND. This mistake has caused data loss in production.

- **Changing the primary key before REWRITE in sequential access mode.** When an indexed file is opened with sequential access, the primary key must not change between the READ and the REWRITE. Changing it causes status 21. In random or dynamic access mode, the primary key identifies the record, so it similarly must not change.

- **Attempting DELETE on a sequential file.** DELETE is only valid for indexed and relative files. Sequential files do not support record deletion. To "delete" records from a sequential file, you must copy the file while skipping the unwanted records.

- **Not reading before REWRITE or DELETE in sequential access mode.** In sequential access mode, REWRITE and DELETE operate on the record most recently read. If the last I/O operation was not a successful READ, file status 43 results. In random or dynamic access mode, the record is identified by the key value, and no prior READ is needed.

- **Using HIGH-VALUES as an EOF sentinel in match-merge without care.** In match-merge programs, setting the key to HIGH-VALUES when a file reaches end-of-file is a common technique. However, if the key is numeric (COMP or COMP-3), moving HIGH-VALUES to it produces invalid numeric data and potential abends. Use a separate EOF flag instead or ensure the key is alphanumeric.

- **Forgetting READ NEXT in DYNAMIC access mode.** After a random READ or START, sequential retrieval must use `READ file NEXT` rather than plain `READ file`. A plain READ in dynamic access mode performs a random read using the current key value, not a sequential read to the next record.

- **File status 39 on OPEN due to attribute mismatch.** This occurs when the record length, key length, or key position defined in the program does not match the actual file. On mainframes, verify that the VSAM cluster definition matches the COBOL FD and SELECT. This is a frequent problem after copybook changes when the file is not reloaded.

- **Variable-length record handling mistakes.** When using RECORD IS VARYING, failing to set the DEPENDING ON field before a WRITE results in an incorrect record length being written. When reading, the system sets the field automatically, but using it after end-of-file may give stale values.

- **Closing a file twice or not closing at all.** Closing a file that is not open produces status 42. Not closing a file may cause data loss, as output buffers might not be flushed. Always close every file you open, and do so exactly once.

- **OPTIONAL clause omission for files that may not exist.** If a program opens a file INPUT that might not exist, the OPEN fails with status 35. Adding `SELECT OPTIONAL file-name` in the FILE-CONTROL paragraph allows the OPEN to succeed with status 05, treating the file as empty. This is commonly needed for restart/checkpoint files.

## Related Topics

- [vsam.md](vsam.md) -- VSAM is the underlying access method for indexed and relative files on mainframes. VSAM KSDS maps to COBOL indexed organization, VSAM RRDS maps to relative organization, and VSAM ESDS maps to sequential organization. Understanding VSAM cluster definitions is essential for mainframe file handling.
- [error_handling.md](error_handling.md) -- File status checking is a critical part of COBOL error handling strategy. Covers broader patterns for error detection, reporting, and recovery including file I/O error handling.
- [jcl_interaction.md](jcl_interaction.md) -- On mainframes, the ASSIGN TO clause in SELECT maps to DD statements in JCL. JCL controls file allocation, disposition, space, and DCB attributes that directly affect file handling behavior.
- [data_types.md](data_types.md) -- Understanding COBOL data types is essential for defining record layouts in FD entries, key fields for indexed files, and the FILE STATUS field.
- [batch_patterns.md](batch_patterns.md) -- Batch processing patterns such as match-merge, sequential update, and master file maintenance are built on top of file handling operations.
- [cobol_structure.md](cobol_structure.md) -- The ENVIRONMENT DIVISION (FILE-CONTROL paragraph) and DATA DIVISION (FILE SECTION) are integral parts of COBOL program structure where file definitions reside.
