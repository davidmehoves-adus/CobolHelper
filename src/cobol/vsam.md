# VSAM

## Description

VSAM (Virtual Storage Access Method) is IBM's high-performance file management system for organizing, storing, and retrieving data on direct-access storage devices (DASD) in mainframe environments. This file covers the four VSAM dataset organizations (KSDS, ESDS, RRDS, VRRDS), SELECT/ASSIGN syntax, access modes (SEQUENTIAL, RANDOM, DYNAMIC), I/O verbs (READ, WRITE, REWRITE, DELETE, START), VSAM file status codes, alternate indexes, and performance considerations including CI/CA splits. Reference this file when working with VSAM datasets in COBOL or diagnosing VSAM errors.

## Table of Contents

- [Core Concepts](#core-concepts)
  - [VSAM Dataset Organizations](#vsam-dataset-organizations)
  - [Control Intervals and Control Areas](#control-intervals-and-control-areas)
  - [Access Modes](#access-modes)
  - [Alternate Indexes](#alternate-indexes)
- [Syntax & Examples](#syntax--examples)
  - [SELECT/ASSIGN for VSAM Files](#selectassign-for-vsam-files)
  - [FD and Record Definitions](#fd-and-record-definitions)
  - [READ Statement with VSAM](#read-statement-with-vsam)
  - [WRITE Statement with VSAM](#write-statement-with-vsam)
  - [REWRITE Statement with VSAM](#rewrite-statement-with-vsam)
  - [DELETE Statement with VSAM](#delete-statement-with-vsam)
  - [START Statement with VSAM](#start-statement-with-vsam)
  - [OPEN and CLOSE](#open-and-close)
- [Common Patterns](#common-patterns)
  - [Sequential Processing of a KSDS](#sequential-processing-of-a-ksds)
  - [Random Lookup by Key](#random-lookup-by-key)
  - [Dynamic Access: Skip-Sequential Processing](#dynamic-access-skip-sequential-processing)
  - [Batch Update of a KSDS](#batch-update-of-a-ksds)
  - [Processing via Alternate Index](#processing-via-alternate-index)
  - [ESDS Append-Only Log Pattern](#esds-append-only-log-pattern)
  - [RRDS Direct Slot Access](#rrds-direct-slot-access)
- [Gotchas](#gotchas)
- [VSAM File Status Codes](#vsam-file-status-codes)
- [Performance Considerations](#performance-considerations)
- [Related Topics](#related-topics)

## Core Concepts

### VSAM Dataset Organizations

VSAM supports four dataset organizations, each suited to different access patterns:

**KSDS (Key-Sequenced Data Set)**

KSDS is the most commonly used VSAM organization. Records are stored in logical key sequence and accessed by a primary key defined during cluster creation. KSDS maintains an index component and a data component. The index component contains pointers that map key values to physical record locations in the data component. Records can be variable-length or fixed-length. KSDS supports all access modes (SEQUENTIAL, RANDOM, DYNAMIC) and all I/O operations (READ, WRITE, REWRITE, DELETE, START). When a record is inserted, VSAM places it in key sequence, performing CI or CA splits if necessary to maintain order.

In COBOL, a KSDS is defined with `ORGANIZATION IS INDEXED` in the SELECT statement.

**ESDS (Entry-Sequenced Data Set)**

ESDS stores records in the order they are written, similar to a sequential file. Records are identified by their Relative Byte Address (RBA) -- the byte offset from the beginning of the dataset. New records are always appended to the end; records cannot be deleted, and their length cannot change on a REWRITE (the RBA must remain stable). ESDS is commonly used for log files, audit trails, and journal records where chronological order matters.

In COBOL, an ESDS is defined with `ORGANIZATION IS SEQUENTIAL` in the SELECT statement. ESDS supports READ and WRITE (append) operations. REWRITE is permitted but the record length must not change. DELETE is not permitted.

**RRDS (Relative Record Data Set) -- Fixed-Length**

RRDS stores fixed-length records in numbered slots. Each slot is identified by a Relative Record Number (RRN), starting at 1. Slots can be empty or occupied. RRDS provides extremely fast direct access by slot number, making it ideal for lookup tables, hash-based access, or any scenario where records map to a numeric identifier. Records can be inserted into any empty slot and deleted (which empties the slot).

In COBOL, an RRDS is defined with `ORGANIZATION IS RELATIVE` in the SELECT statement, using `RELATIVE KEY IS` to specify the data item holding the RRN.

**VRRDS (Variable-Length Relative Record Data Set)**

VRRDS is a variation of RRDS that allows variable-length records. Like RRDS, records are accessed by relative record number, but each slot can hold records of different sizes up to the maximum defined for the cluster. VRRDS is less commonly used than RRDS but provides flexibility when record sizes vary but slot-based access is still desired.

In COBOL, VRRDS is coded identically to RRDS (`ORGANIZATION IS RELATIVE`); the variable-length nature is a property of the VSAM cluster definition (IDCAMS), not the COBOL program.

### Control Intervals and Control Areas

VSAM organizes data into **Control Intervals (CIs)** and **Control Areas (CAs)**:

- A **Control Interval** is the unit of data transfer between DASD and a VSAM buffer. A CI contains one or more logical records, free space, and control information (Record Definition Fields and CI Definition Field). CI size is set during cluster definition and typically ranges from 512 to 32,768 bytes.

- A **Control Area** is a group of control intervals. A CA is the unit of space allocation for VSAM. When a KSDS is defined, VSAM allocates space in CA-sized units.

- **Free Space**: During KSDS cluster definition, you can specify free space as a percentage of each CI and a percentage of CAs to leave entirely free. For example, `FREESPACE(20 10)` reserves 20% free space in each CI and leaves 10% of CAs completely empty. This free space accommodates future insertions without immediate splits.

- **CI Split**: When a record must be inserted into a CI that has no room, VSAM performs a CI split. Approximately half the records in the full CI are moved to a free CI within the same CA, and the new record is inserted in its proper key-sequence position. The index is updated to reflect the new arrangement.

- **CA Split**: When a CI split is needed but there are no free CIs in the current CA, VSAM performs a CA split. A new CA is allocated, approximately half the CIs from the original CA are moved to the new CA, and the index is updated. CA splits are significantly more expensive than CI splits because they involve moving much more data and updating higher-level index records.

### Access Modes

COBOL supports three access modes for VSAM files, specified in the SELECT statement:

**SEQUENTIAL**

Records are accessed in sequence -- by key order for KSDS, by entry order for ESDS, or by RRN order for RRDS. This is the default if ACCESS MODE is not specified. Sequential access is the most efficient mode for processing all or most records in a file. With KSDS, sequential access follows the primary key index. The START verb can be used to position within the file before beginning sequential reads.

**RANDOM**

Records are accessed directly by key (KSDS), by RRN (RRDS/VRRDS), or are not applicable to ESDS in standard COBOL usage. Each I/O operation specifies the target record through the RECORD KEY or RELATIVE KEY. Random access is optimal when only a small percentage of records need to be accessed and their keys are known.

**DYNAMIC**

Dynamic access combines sequential and random access in the same OPEN scope. The program can switch between reading sequentially (READ NEXT) and accessing randomly (READ with key) without closing and reopening the file. Dynamic access is essential for skip-sequential processing, where you position to a starting key with START, read sequentially through a range, then reposition with another START or random READ.

### Alternate Indexes

An **alternate index (AIX)** provides an additional access path into a KSDS or ESDS based on a field other than the primary key. For example, a customer file keyed by customer number might have an alternate index on customer name.

Key characteristics:
- An AIX is itself a KSDS whose records contain the alternate key and one or more primary key pointers.
- Alternate keys can be **unique** (each alternate key value maps to exactly one base record) or **non-unique** (one alternate key value maps to multiple base records).
- A **path** is defined over the AIX to connect it to the base cluster. The COBOL program references the path, not the AIX directly.
- Multiple AIXs can be defined for a single base cluster.
- AIXs must be built (using IDCAMS BLDINDEX) after the base cluster is loaded and must be kept in sync. When the base cluster is updated, the associated AIXs should be updated. If the base cluster is opened for output and the AIX is defined in a VSAM upgrade set, VSAM automatically maintains the AIX.
- Each AIX path is referenced by a separate SELECT/ASSIGN in the COBOL program and has its own DD/DLBL statement in JCL.

## Syntax & Examples

### SELECT/ASSIGN for VSAM Files

**KSDS SELECT:**

```cobol
       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT CUSTOMER-FILE
               ASSIGN TO CUSTFILE
               ORGANIZATION IS INDEXED
               ACCESS MODE IS DYNAMIC
               RECORD KEY IS CUST-KEY
               ALTERNATE RECORD KEY IS CUST-NAME
                   WITH DUPLICATES
               FILE STATUS IS WS-FILE-STATUS.
```

- `ASSIGN TO CUSTFILE` -- the ddname in JCL that points to the VSAM cluster.
- `ORGANIZATION IS INDEXED` -- indicates KSDS.
- `ACCESS MODE IS DYNAMIC` -- allows both sequential and random access.
- `RECORD KEY IS CUST-KEY` -- the primary key field, must match the key defined in the VSAM cluster.
- `ALTERNATE RECORD KEY IS CUST-NAME WITH DUPLICATES` -- an alternate index path allowing non-unique keys.
- `FILE STATUS IS WS-FILE-STATUS` -- a two-byte field to receive the result of each I/O operation.

**ESDS SELECT:**

```cobol
           SELECT AUDIT-LOG
               ASSIGN TO AUDLOG
               ORGANIZATION IS SEQUENTIAL
               ACCESS MODE IS SEQUENTIAL
               FILE STATUS IS WS-AUDIT-STATUS.
```

ESDS files use `ORGANIZATION IS SEQUENTIAL`. Only sequential access is available in standard COBOL for ESDS.

**RRDS SELECT:**

```cobol
           SELECT SLOT-TABLE
               ASSIGN TO SLOTTBL
               ORGANIZATION IS RELATIVE
               ACCESS MODE IS RANDOM
               RELATIVE KEY IS WS-SLOT-NUMBER
               FILE STATUS IS WS-SLOT-STATUS.
```

- `ORGANIZATION IS RELATIVE` -- indicates RRDS or VRRDS.
- `RELATIVE KEY IS WS-SLOT-NUMBER` -- a numeric data item holding the relative record number (starting at 1). This field must NOT be part of the record itself; it must be a WORKING-STORAGE item.

### FD and Record Definitions

```cobol
       DATA DIVISION.
       FILE SECTION.
       FD  CUSTOMER-FILE.
       01  CUSTOMER-RECORD.
           05 CUST-KEY              PIC X(10).
           05 CUST-NAME             PIC X(30).
           05 CUST-ADDRESS          PIC X(50).
           05 CUST-BALANCE          PIC S9(7)V99 COMP-3.

       WORKING-STORAGE SECTION.
       01  WS-FILE-STATUS           PIC X(02).
       01  WS-SLOT-NUMBER           PIC 9(08) COMP.
```

The `RECORD KEY IS CUST-KEY` in the SELECT must reference a field defined in the FD's 01-level record. The key field's position and length must match the key defined when the VSAM cluster was created via IDCAMS DEFINE CLUSTER.

### READ Statement with VSAM

**Sequential READ (KSDS, ESDS, RRDS):**

```cobol
           READ CUSTOMER-FILE NEXT RECORD
               INTO WS-CUSTOMER-RECORD
               AT END
                   SET WS-END-OF-FILE TO TRUE
               NOT AT END
                   PERFORM PROCESS-CUSTOMER
           END-READ
```

The `NEXT` keyword is required when ACCESS MODE IS DYNAMIC to distinguish sequential reads from random reads. For ACCESS MODE IS SEQUENTIAL, `NEXT` is optional but recommended for clarity.

**Random READ (KSDS):**

```cobol
           MOVE '0000012345' TO CUST-KEY
           READ CUSTOMER-FILE
               INTO WS-CUSTOMER-RECORD
               KEY IS CUST-KEY
               INVALID KEY
                   DISPLAY 'CUSTOMER NOT FOUND'
               NOT INVALID KEY
                   PERFORM PROCESS-CUSTOMER
           END-READ
```

For random reads, move the desired key value into the RECORD KEY field before the READ. The `KEY IS` phrase is optional when reading by the primary key but is required when reading by an alternate key.

**Random READ (RRDS):**

```cobol
           MOVE 42 TO WS-SLOT-NUMBER
           READ SLOT-TABLE
               INTO WS-SLOT-RECORD
               INVALID KEY
                   DISPLAY 'SLOT EMPTY OR NOT FOUND'
               NOT INVALID KEY
                   PERFORM PROCESS-SLOT
           END-READ
```

### WRITE Statement with VSAM

**WRITE to KSDS (INSERT):**

```cobol
           MOVE '0000099999'  TO CUST-KEY
           MOVE 'NEW CUSTOMER' TO CUST-NAME
           MOVE SPACES          TO CUST-ADDRESS
           MOVE ZEROS           TO CUST-BALANCE

           WRITE CUSTOMER-RECORD
               INVALID KEY
                   DISPLAY 'DUPLICATE KEY - WRITE FAILED'
               NOT INVALID KEY
                   DISPLAY 'RECORD ADDED SUCCESSFULLY'
           END-WRITE
```

For KSDS, WRITE inserts a new record in key sequence. The INVALID KEY condition fires if a record with the same primary key already exists (status code 22) or if an alternate unique key is duplicated.

**WRITE to ESDS (APPEND):**

```cobol
           WRITE AUDIT-RECORD
               FROM WS-AUDIT-DATA
           END-WRITE
```

ESDS WRITE always appends to the end of the dataset. There is no INVALID KEY condition for ESDS writes (no key to collide with), though file status should still be checked.

**WRITE to RRDS:**

```cobol
           MOVE 100 TO WS-SLOT-NUMBER
           WRITE SLOT-RECORD
               INVALID KEY
                   DISPLAY 'SLOT ALREADY OCCUPIED'
               NOT INVALID KEY
                   DISPLAY 'RECORD WRITTEN TO SLOT'
           END-WRITE
```

For RRDS, the record is written to the slot identified by the RELATIVE KEY. INVALID KEY fires if the slot is already occupied.

### REWRITE Statement with VSAM

```cobol
           READ CUSTOMER-FILE
               INTO WS-CUSTOMER-RECORD
               KEY IS CUST-KEY
               INVALID KEY
                   DISPLAY 'NOT FOUND'
           END-READ

           IF WS-FILE-STATUS = '00'
               ADD 100.00 TO CUST-BALANCE
               REWRITE CUSTOMER-RECORD
                   FROM WS-CUSTOMER-RECORD
                   INVALID KEY
                       DISPLAY 'REWRITE FAILED'
                   NOT INVALID KEY
                       DISPLAY 'RECORD UPDATED'
               END-REWRITE
           END-IF
```

Important rules for REWRITE:
- For KSDS with ACCESS MODE IS SEQUENTIAL, a successful READ must immediately precede REWRITE (no intervening I/O on the same file). The primary key must not be changed.
- For KSDS with ACCESS MODE IS RANDOM or DYNAMIC, a prior READ is not required for REWRITE; the record is located by its key value.
- For ESDS, REWRITE is allowed but the record length must not change.
- For RRDS, the record is rewritten to the same slot.
- The primary key of a KSDS record cannot be changed via REWRITE. To change a key, DELETE the old record and WRITE a new one.

### DELETE Statement with VSAM

```cobol
           MOVE '0000012345' TO CUST-KEY
           DELETE CUSTOMER-FILE RECORD
               INVALID KEY
                   DISPLAY 'RECORD NOT FOUND FOR DELETE'
               NOT INVALID KEY
                   DISPLAY 'RECORD DELETED'
           END-DELETE
```

DELETE rules:
- DELETE is valid only for KSDS and RRDS/VRRDS. It is **not** valid for ESDS.
- For ACCESS MODE IS SEQUENTIAL, a successful READ must precede the DELETE (the current record is deleted). The `INVALID KEY` phrase is not used in this case.
- For ACCESS MODE IS RANDOM or DYNAMIC, the key or RRN identifies the record to delete. No prior READ is needed, and the `INVALID KEY` phrase is used.

### START Statement with VSAM

```cobol
           MOVE 'M' TO CUST-KEY
           START CUSTOMER-FILE
               KEY IS GREATER THAN OR EQUAL TO CUST-KEY
               INVALID KEY
                   DISPLAY 'NO RECORDS FROM THIS POINT'
               NOT INVALID KEY
                   DISPLAY 'POSITIONED SUCCESSFULLY'
           END-START
```

START positions the file pointer for subsequent sequential READs without actually reading a record. It is valid for KSDS and RRDS (not meaningful for ESDS in standard usage).

Supported relational operators:
- `KEY IS EQUAL TO`
- `KEY IS GREATER THAN`
- `KEY IS NOT LESS THAN` (equivalent to GREATER THAN OR EQUAL TO)
- `KEY IS GREATER THAN OR EQUAL TO`

A common technique is to use a partial key with START. If `CUST-KEY` is PIC X(10) but you move only one character and use a reference-modified key or a shorter field, you can position to the first record matching a prefix.

### OPEN and CLOSE

```cobol
           OPEN I-O CUSTOMER-FILE
           IF WS-FILE-STATUS NOT = '00'
               DISPLAY 'OPEN FAILED: ' WS-FILE-STATUS
               STOP RUN
           END-IF

      *    ... process records ...

           CLOSE CUSTOMER-FILE
           IF WS-FILE-STATUS NOT = '00'
               DISPLAY 'CLOSE FAILED: ' WS-FILE-STATUS
           END-IF
```

VSAM files support these OPEN modes:
- `OPEN INPUT` -- read-only access.
- `OPEN OUTPUT` -- write-only; for KSDS/RRDS this implies the file is empty (initial load). Opening a non-empty KSDS as OUTPUT is an error in some environments.
- `OPEN I-O` -- read and write access (READ, WRITE, REWRITE, DELETE).
- `OPEN EXTEND` -- append mode (ESDS only in standard usage; appends records after the last existing record).

## Common Patterns

### Sequential Processing of a KSDS

```cobol
       PROCEDURE DIVISION.
       MAIN-PROCESS.
           OPEN INPUT CUSTOMER-FILE
           IF WS-FILE-STATUS NOT = '00'
               DISPLAY 'OPEN ERROR: ' WS-FILE-STATUS
               STOP RUN
           END-IF

           PERFORM READ-NEXT-CUSTOMER
           PERFORM UNTIL WS-END-OF-FILE
               PERFORM PROCESS-CUSTOMER
               PERFORM READ-NEXT-CUSTOMER
           END-PERFORM

           CLOSE CUSTOMER-FILE
           STOP RUN
           .

       READ-NEXT-CUSTOMER.
           READ CUSTOMER-FILE NEXT RECORD
               INTO WS-CUSTOMER-RECORD
               AT END
                   SET WS-END-OF-FILE TO TRUE
           END-READ
           .
```

This is the standard "read-ahead" loop pattern. The first READ primes the loop, and subsequent READs occur at the end of each iteration.

### Random Lookup by Key

```cobol
       RANDOM-LOOKUP.
           OPEN INPUT CUSTOMER-FILE
           IF WS-FILE-STATUS NOT = '00'
               DISPLAY 'OPEN ERROR: ' WS-FILE-STATUS
               STOP RUN
           END-IF

           MOVE WS-SEARCH-KEY TO CUST-KEY
           READ CUSTOMER-FILE
               INTO WS-CUSTOMER-RECORD
               INVALID KEY
                   MOVE 'N' TO WS-FOUND-FLAG
               NOT INVALID KEY
                   MOVE 'Y' TO WS-FOUND-FLAG
           END-READ

           CLOSE CUSTOMER-FILE
           .
```

### Dynamic Access: Skip-Sequential Processing

The program positions to a starting point, reads a range of records sequentially, then repositions to a different point.

```cobol
       SKIP-SEQUENTIAL.
           OPEN INPUT CUSTOMER-FILE

      *    Position to first record with key >= 'C'
           MOVE 'C' TO CUST-KEY
           START CUSTOMER-FILE
               KEY IS NOT LESS THAN CUST-KEY
               INVALID KEY
                   DISPLAY 'NO RECORDS FROM C'
                   GO TO SKIP-SEQ-EXIT
           END-START

      *    Read sequentially until key no longer starts with 'C'
           READ CUSTOMER-FILE NEXT RECORD
               AT END
                   GO TO SKIP-SEQ-EXIT
           END-READ

           PERFORM UNTIL WS-FILE-STATUS NOT = '00'
                       OR CUST-KEY(1:1) NOT = 'C'
               PERFORM PROCESS-CUSTOMER
               READ CUSTOMER-FILE NEXT RECORD
                   AT END
                       CONTINUE
               END-READ
           END-PERFORM

       SKIP-SEQ-EXIT.
           CLOSE CUSTOMER-FILE
           .
```

### Batch Update of a KSDS

A typical batch job reads a transaction file and applies updates to a VSAM master file:

```cobol
       BATCH-UPDATE.
           OPEN I-O CUSTOMER-FILE
           OPEN INPUT TRANSACTION-FILE

           READ TRANSACTION-FILE
               AT END SET WS-END-OF-TRANS TO TRUE
           END-READ

           PERFORM UNTIL WS-END-OF-TRANS
               EVALUATE TRANS-ACTION
                   WHEN 'A'
                       PERFORM ADD-CUSTOMER
                   WHEN 'U'
                       PERFORM UPDATE-CUSTOMER
                   WHEN 'D'
                       PERFORM DELETE-CUSTOMER
                   WHEN OTHER
                       PERFORM LOG-BAD-TRANSACTION
               END-EVALUATE

               READ TRANSACTION-FILE
                   AT END SET WS-END-OF-TRANS TO TRUE
               END-READ
           END-PERFORM

           CLOSE CUSTOMER-FILE
           CLOSE TRANSACTION-FILE
           .

       ADD-CUSTOMER.
           MOVE TRANS-DATA TO CUSTOMER-RECORD
           WRITE CUSTOMER-RECORD
               INVALID KEY
                   DISPLAY 'DUP KEY ON ADD: ' CUST-KEY
           END-WRITE
           .

       UPDATE-CUSTOMER.
           MOVE TRANS-KEY TO CUST-KEY
           READ CUSTOMER-FILE
               INVALID KEY
                   DISPLAY 'NOT FOUND FOR UPDATE: ' CUST-KEY
           END-READ
           IF WS-FILE-STATUS = '00'
               MOVE TRANS-DATA TO CUSTOMER-RECORD
               REWRITE CUSTOMER-RECORD
                   INVALID KEY
                       DISPLAY 'REWRITE FAILED: ' CUST-KEY
               END-REWRITE
           END-IF
           .

       DELETE-CUSTOMER.
           MOVE TRANS-KEY TO CUST-KEY
           DELETE CUSTOMER-FILE RECORD
               INVALID KEY
                   DISPLAY 'NOT FOUND FOR DELETE: ' CUST-KEY
           END-DELETE
           .
```

### Processing via Alternate Index

```cobol
       FILE-CONTROL.
           SELECT CUST-BY-NAME
               ASSIGN TO CUSTNAME
               ORGANIZATION IS INDEXED
               ACCESS MODE IS DYNAMIC
               RECORD KEY IS CN-CUST-KEY
               FILE STATUS IS WS-AIX-STATUS.

       FD  CUST-BY-NAME.
       01  CN-CUSTOMER-RECORD.
           05 CN-CUST-KEY           PIC X(10).
           05 CN-CUST-NAME          PIC X(30).
           05 CN-CUST-ADDRESS       PIC X(50).
           05 CN-CUST-BALANCE       PIC S9(7)V99 COMP-3.

       PROCEDURE DIVISION.
       SEARCH-BY-NAME.
           OPEN INPUT CUST-BY-NAME
           MOVE WS-SEARCH-NAME TO CN-CUST-NAME
           START CUST-BY-NAME
               KEY IS EQUAL TO CN-CUST-NAME
               INVALID KEY
                   DISPLAY 'NAME NOT FOUND'
           END-START

           IF WS-AIX-STATUS = '00'
               READ CUST-BY-NAME NEXT RECORD
                   AT END CONTINUE
               END-READ
               PERFORM UNTIL WS-AIX-STATUS NOT = '00'
                           OR CN-CUST-NAME NOT = WS-SEARCH-NAME
                   PERFORM DISPLAY-CUSTOMER
                   READ CUST-BY-NAME NEXT RECORD
                       AT END CONTINUE
                   END-READ
               END-PERFORM
           END-IF

           CLOSE CUST-BY-NAME
           .
```

When processing through an alternate index path, the JCL DD statement points to the VSAM path (not the base cluster or AIX directly). The record layout is the same as the base cluster record because VSAM retrieves the full base record through the path.

### ESDS Append-Only Log Pattern

```cobol
       FILE-CONTROL.
           SELECT AUDIT-LOG
               ASSIGN TO AUDLOG
               ORGANIZATION IS SEQUENTIAL
               ACCESS MODE IS SEQUENTIAL
               FILE STATUS IS WS-AUDIT-STATUS.

       PROCEDURE DIVISION.
       WRITE-AUDIT-ENTRY.
           OPEN EXTEND AUDIT-LOG
           IF WS-AUDIT-STATUS NOT = '00'
               DISPLAY 'AUDIT LOG OPEN FAILED'
               STOP RUN
           END-IF

           MOVE CURRENT-DATE TO AUDIT-TIMESTAMP
           MOVE WS-USER-ID   TO AUDIT-USER
           MOVE WS-ACTION    TO AUDIT-ACTION
           MOVE WS-DETAIL    TO AUDIT-DETAIL

           WRITE AUDIT-RECORD
           IF WS-AUDIT-STATUS NOT = '00'
               DISPLAY 'AUDIT WRITE FAILED: ' WS-AUDIT-STATUS
           END-IF

           CLOSE AUDIT-LOG
           .
```

### RRDS Direct Slot Access

```cobol
       FILE-CONTROL.
           SELECT RATE-TABLE
               ASSIGN TO RATETBL
               ORGANIZATION IS RELATIVE
               ACCESS MODE IS RANDOM
               RELATIVE KEY IS WS-RATE-SLOT
               FILE STATUS IS WS-RATE-STATUS.

       WORKING-STORAGE SECTION.
       01  WS-RATE-SLOT             PIC 9(04) COMP.

       PROCEDURE DIVISION.
       LOOKUP-RATE.
           OPEN INPUT RATE-TABLE
           MOVE WS-RATE-CODE TO WS-RATE-SLOT
           READ RATE-TABLE
               INTO WS-RATE-RECORD
               INVALID KEY
                   DISPLAY 'RATE NOT FOUND FOR CODE: '
                       WS-RATE-CODE
           END-READ
           CLOSE RATE-TABLE
           .
```

RRDS is ideal for lookup tables where the key maps directly to a slot number, providing single-I/O retrieval.

## Gotchas

- **RELATIVE KEY must be in WORKING-STORAGE**: For RRDS, the RELATIVE KEY data item must be defined in WORKING-STORAGE (or LOCAL-STORAGE), never in the FD record area. The compiler will reject a RELATIVE KEY field defined in the file's record description. This is different from RECORD KEY for KSDS, which must be in the FD.

- **RECORD KEY must be in the FD record area**: For KSDS, the RECORD KEY field must be defined within the 01-level record under the FD. Defining it in WORKING-STORAGE is a compilation error. The field's offset and length must match the key defined in the VSAM cluster.

- **OPEN OUTPUT on a non-empty KSDS deletes all records**: Opening a KSDS with OPEN OUTPUT treats it as an initial load -- existing records may be deleted. Use OPEN I-O for updates to existing datasets. This is one of the most dangerous mistakes in VSAM programming.

- **Sequential REWRITE/DELETE requires a prior READ**: When ACCESS MODE IS SEQUENTIAL, REWRITE and DELETE operate on the record most recently read. If no READ has been performed, or if an intervening I/O operation on the same file has occurred, the REWRITE or DELETE will fail. With RANDOM or DYNAMIC access, no prior READ is needed for REWRITE or DELETE.

- **READ NEXT vs READ in DYNAMIC mode**: In ACCESS MODE IS DYNAMIC, a plain `READ filename` is a random read by key. To read the next sequential record, you must code `READ filename NEXT`. Omitting NEXT when you intended sequential processing will cause the program to repeatedly read the same record (the one matching the current key value).

- **Alternate index WITH DUPLICATES -- ordering is not guaranteed across updates**: When an alternate index allows duplicates, the order in which duplicate-key records are returned during sequential processing is generally the order of their primary keys, but this can be disrupted by updates, splits, and reorganizations.

- **CA splits degrade performance silently**: CA splits do not produce error codes or warnings visible to the COBOL program. Performance degrades gradually as splits create out-of-sequence CIs that require additional I/O. Monitor CA split activity through IDCAMS LISTCAT statistics, not from within the COBOL program.

- **File status must always be checked**: Every VSAM I/O operation should be followed by a file status check. Relying solely on INVALID KEY misses errors like I/O errors (status 9x), file-not-found on OPEN (status 35), and other conditions that do not trigger INVALID KEY.

- **ESDS records cannot be deleted**: There is no DELETE operation for ESDS. If you need to logically remove records, use a flag field in the record and filter on that flag when reading.

- **ESDS REWRITE cannot change record length**: For ESDS, REWRITE must preserve the original record length exactly. Changing the length of an ESDS record will result in a file status error because the RBA-based addressing requires records to stay at their original positions.

- **VSAM empty file on first OPEN INPUT returns status 35**: If a VSAM file has never had any records written to it, opening it for INPUT may return status 35 (file not found) even though the cluster has been defined. The cluster must be loaded (at least one record written via OUTPUT mode) before it can be opened for INPUT or I-O. Alternatively, define the cluster with RECORDS(0) or use an initial-load step.

- **Key field content mismatch with cluster definition**: If the COBOL program's RECORD KEY field offset or length does not match the key offset and length defined in the VSAM cluster (via IDCAMS DEFINE CLUSTER KEYS parameter), the program may compile cleanly but produce incorrect results or abend at runtime. Always verify the key definition matches.

- **START does not read a record**: START only positions the file pointer; it does not return a record. After a successful START, you must issue a READ NEXT to retrieve the first record in the range.

- **Mixed I/O on the same file without proper sequencing**: In ACCESS MODE IS DYNAMIC, switching between random and sequential operations requires care. After a random READ, a subsequent READ NEXT will read the record sequentially after the one just randomly read. After a START, READ NEXT reads from the START position. Misunderstanding this sequencing leads to skipped or repeated records.

## VSAM File Status Codes

VSAM uses the standard two-byte file status code plus extended VSAM status codes. Standard file status codes (00-49) apply to VSAM files with the same meanings as for any COBOL file. See [file_handling.md](file_handling.md) for the complete file status code reference table.

The 9x range is implementor-defined and has VSAM-specific meanings:

| Status | Meaning |
|--------|---------|
| 90 | General VSAM logic error (consult the extended status code) |
| 91 | VSAM password failure |
| 92 | Logic error; for example, OPEN with conflicting filename |
| 93 | VSAM resource not available (e.g., insufficient virtual storage) |
| 94 | Sequential READ after the current record pointer is undefined |
| 95 | Invalid or incomplete VSAM file information |
| 96 | No DD statement found for the file |
| 97 | OPEN successful; file integrity verified |

For status codes 9x, the full VSAM return code and reason code are available in an extended file status area if the program defines one. This is done by specifying a 6-byte (or longer) file status field where bytes 3-6 contain the VSAM feedback code, providing more detail for debugging.

## Performance Considerations

**CI Size Selection**

Choose CI size based on access pattern. For sequential processing, larger CIs (e.g., 4096, 8192, or higher) reduce the number of I/O operations because more records are read per CI. For random processing, smaller CIs reduce the amount of wasted data transferred per I/O. A common compromise for mixed workloads is 4096 bytes.

**Free Space Planning**

Allocate free space thoughtfully for KSDS files that receive insertions:
- **CI free space** (e.g., 10-30%): Reserves space within each CI for new records, reducing CI splits.
- **CA free space** (e.g., 5-15%): Reserves entire free CIs within each CA, providing overflow space when CI splits do occur, reducing CA splits.
- Over-allocating wastes DASD and increases sequential processing time. Under-allocating causes excessive splits. Files loaded once need little free space; files with heavy inserts need more.

**CI and CA Splits**

- **CI splits** cause moderate overhead: records are moved within the same CA, and the index is updated. Occasional CI splits are normal and acceptable.
- **CA splits** are expensive: a new CA is allocated, half the CIs from the old CA are moved, and multiple levels of index are updated. Frequent CA splits indicate the free space allocation is insufficient or the file needs reorganization.
- Monitor split statistics with `IDCAMS LISTCAT ALL` -- check CI-SPLITS and CA-SPLITS counters. A rising CA split count is a signal to reorganize.
- **Reorganization** (IDCAMS REPRO to unload, DELETE/DEFINE to recreate, REPRO to reload) resets free space and eliminates split-related fragmentation. Schedule reorganizations during maintenance windows based on split activity.

**Buffering**

- VSAM uses data buffers (BUFND) and index buffers (BUFNI). Increasing buffer counts can significantly improve performance by caching more CIs in memory.
- For sequential processing, additional data buffers (BUFND) allow VSAM to read ahead, reducing physical I/O.
- For random processing, additional index buffers (BUFNI) keep higher-level index records in memory, reducing the number of I/O operations needed to locate a record.
- Buffers can be specified in JCL (AMP parameter), in IDCAMS definitions, or as installation defaults.

**Sequential vs. Random I/O Efficiency**

- Sequential access to a KSDS is nearly as efficient as reading a sequential file, because VSAM reads CIs in order and benefits from read-ahead buffering.
- Random access requires traversing the index (sequence set and possibly higher-level index sets) for each read, making it more I/O-intensive per record.
- If you must process more than about 10-20% of the records in a KSDS, sequential processing is almost always faster than random access, even if it means reading and skipping unwanted records.

**Alternate Index Overhead**

- Each alternate index adds overhead to WRITE, REWRITE, and DELETE operations because the AIX must be updated to stay in sync with the base cluster.
- For batch jobs that do heavy updates, consider temporarily disconnecting AIXs from the upgrade set and rebuilding them (BLDINDEX) after the batch run completes.

## Related Topics

- **[file_handling.md](file_handling.md)** -- Covers general COBOL file handling concepts (OPEN, CLOSE, READ, WRITE) for all file organizations. VSAM uses the same I/O verbs but with VSAM-specific behavior and status codes described in this file.
- **[error_handling.md](error_handling.md)** -- Covers file status checking patterns and error-handling strategies. VSAM file status codes (especially the 9x range) require specialized error handling described here.
- **[jcl_interaction.md](jcl_interaction.md)** -- Covers JCL DD statements and the ASSIGN clause. VSAM files require specific JCL (DD for clusters, paths, AMP parameters for buffering) that connect to the COBOL SELECT/ASSIGN.
- **[data_types.md](data_types.md)** -- Covers COBOL data types and PICTURE clauses. VSAM key fields and record layouts rely on correct data type definitions, particularly for key fields that must match the cluster definition.
