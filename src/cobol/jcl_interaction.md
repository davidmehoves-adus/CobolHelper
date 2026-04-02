# JCL Interaction

## Description
This file covers the interface between COBOL programs and Job Control Language (JCL), the mechanism by which z/OS batch programs receive their runtime environment. It explains how JCL DD statements map to COBOL file definitions, how parameters are passed to programs, how condition codes flow between job steps, and how datasets are allocated and managed at execution time. Reference this file whenever you need to understand how a COBOL program communicates with its JCL wrapper or when diagnosing runtime issues related to file allocation, parameter passing, or step-level condition code logic.

## Table of Contents
- [Core Concepts](#core-concepts)
  - [The JCL-COBOL Contract](#the-jcl-cobol-contract)
  - [DD Names and SELECT/ASSIGN](#dd-names-and-selectassign)
  - [JCL DD and COBOL File Definitions](#jcl-dd-and-cobol-file-definitions)
  - [EXEC PGM and Program Invocation](#exec-pgm-and-program-invocation)
  - [PARM Field and Accepting Parameters](#parm-field-and-accepting-parameters)
  - [RETURN-CODE and the COND Parameter](#return-code-and-the-cond-parameter)
  - [Condition Code Handling](#condition-code-handling)
  - [SYSIN and SYSOUT](#sysin-and-sysout)
  - [STEPLIB and JOBLIB](#steplib-and-joblib)
  - [Concatenated Datasets](#concatenated-datasets)
  - [GDG Basics for COBOL Programs](#gdg-basics-for-cobol-programs)
- [Syntax & Examples](#syntax--examples)
- [Common Patterns](#common-patterns)
- [Gotchas](#gotchas)
- [Related Topics](#related-topics)

## Core Concepts

### The JCL-COBOL Contract

A COBOL program compiled into a load module does not run in isolation. On z/OS (and its predecessors MVS, OS/390), JCL is the glue that tells the operating system which program to execute, what datasets to allocate, what parameters to pass, and how to interpret the results. The relationship is contractual: the COBOL source declares what external resources it expects (files, parameters), and the JCL provides them at runtime. A mismatch between these two sides is one of the most common sources of batch job failures.

The key touchpoints between JCL and COBOL are:

1. **File allocation** -- JCL DD statements provide physical datasets; COBOL SELECT/ASSIGN clauses reference them by DD name.
2. **Parameter passing** -- The JCL EXEC statement PARM field passes a string to the program; COBOL receives it via ACCEPT or linkage conventions.
3. **Condition codes** -- The COBOL program sets RETURN-CODE; JCL COND parameters or IF/THEN/ELSE constructs test it to control step execution.
4. **Program location** -- STEPLIB or JOBLIB DD statements tell the system where to find the load module.
5. **System files** -- SYSIN provides inline input; SYSOUT captures printed output and messages.

### DD Names and SELECT/ASSIGN

The DD name is the linchpin of JCL-COBOL file communication. In the COBOL source, the ENVIRONMENT DIVISION contains a FILE-CONTROL paragraph where each file is declared with a SELECT statement:

```cobol
ENVIRONMENT DIVISION.
INPUT-OUTPUT SECTION.
FILE-CONTROL.
    SELECT CUSTOMER-FILE
        ASSIGN TO CUSTMAST
        ORGANIZATION IS SEQUENTIAL
        ACCESS MODE IS SEQUENTIAL
        FILE STATUS IS WS-CUST-STATUS.
```

The name after ASSIGN TO -- here `CUSTMAST` -- is the **DD name** that must appear in the JCL:

```jcl
//CUSTMAST DD DSN=PROD.CUSTOMER.MASTER,DISP=SHR
```

The operating system resolves this at OPEN time: when the COBOL program executes `OPEN INPUT CUSTOMER-FILE`, the runtime locates the DD name `CUSTMAST` in the step's allocation table and connects the COBOL file descriptor to the physical dataset `PROD.CUSTOMER.MASTER`.

**ASSIGN TO variations:**

The ASSIGN clause has several forms depending on the compiler and installation defaults:

- `ASSIGN TO CUSTMAST` -- The DD name is `CUSTMAST` (most common on z/OS).
- `ASSIGN TO DDNAME-CUSTMAST` -- Equivalent; the `DDNAME-` prefix is stripped.
- `ASSIGN TO S-CUSTMAST` -- The `S-` prefix indicates sequential; the DD name is `CUSTMAST`.
- `ASSIGN TO AS-CUSTMAST` -- The `AS-` prefix indicates actual sequential; the DD name is `CUSTMAST`.
- `ASSIGN TO CUSTMAST USING WS-DDNAME` -- Dynamic DD name from a working-storage variable (less common).

The critical rule: whatever name the runtime derives from the ASSIGN clause must match a DD statement in the executing JCL step, or the OPEN will fail with a file status of `35` (file not found) or an S013 abend.

### JCL DD and COBOL File Definitions

The DD statement and the COBOL FD (File Description) must agree on certain physical attributes, or unpredictable results occur. The key attributes are:

| Attribute | JCL DD Parameter | COBOL FD Clause |
|-----------|-----------------|-----------------|
| Record format | `RECFM=FB` | `RECORDING MODE IS F` |
| Logical record length | `LRECL=80` | `RECORD CONTAINS 80 CHARACTERS` or 01-level size |
| Block size | `BLKSIZE=8000` | `BLOCK CONTAINS 100 RECORDS` |
| Dataset organization | `DSORG=PS` (sequential) | `ORGANIZATION IS SEQUENTIAL` |

In practice, the system merges attributes from three sources in this priority order:

1. The DD statement in JCL
2. The dataset label (catalog/VTOC)
3. The program (COBOL FD)

If the JCL specifies `LRECL=100` but the COBOL FD implies an 80-byte record, the JCL wins at runtime, which often leads to data truncation or padding without any compile-time warning.

A typical FD and its corresponding DD:

```cobol
DATA DIVISION.
FILE SECTION.
FD  TRANSACTION-FILE
    RECORDING MODE IS F
    BLOCK CONTAINS 0 RECORDS
    RECORD CONTAINS 150 CHARACTERS.
01  TRANSACTION-RECORD.
    05  TXN-TYPE           PIC X(02).
    05  TXN-ACCOUNT        PIC X(10).
    05  TXN-AMOUNT         PIC S9(09)V99 COMP-3.
    05  FILLER             PIC X(132).
```

```jcl
//TXNFILE  DD DSN=PROD.DAILY.TRANSACTIONS,
//            DISP=SHR,
//            DCB=(RECFM=FB,LRECL=150,BLKSIZE=0)
```

Setting `BLOCK CONTAINS 0 RECORDS` in COBOL and `BLKSIZE=0` in JCL tells the system to choose an optimal block size automatically (System-Determined Blocksize, or SDB).

### EXEC PGM and Program Invocation

The JCL EXEC statement launches the COBOL program:

```jcl
//STEP01   EXEC PGM=CUSTUPDT
```

This tells the system to search for load module `CUSTUPDT` and transfer control to it. The search order for the load module is:

1. The **STEPLIB** DD (if present in this step)
2. The **JOBLIB** DD (if present at the job level and no STEPLIB exists for the step)
3. The system link list (LNKLSTxx)
4. The link pack area (LPA)

The program name in `PGM=` corresponds to the member name in the load library (PDS or PDSE) where the compiled and linked COBOL program resides.

An alternative form uses a cataloged or in-stream procedure:

```jcl
//STEP01   EXEC PROC=CUSTPROC
```

or simply:

```jcl
//STEP01   EXEC CUSTPROC
```

Inside the procedure, the actual `PGM=` parameter identifies the COBOL program.

### PARM Field and Accepting Parameters

The EXEC statement can pass a character string to the program via the PARM parameter:

```jcl
//STEP01   EXEC PGM=RPTGEN,PARM='20260101D'
```

The PARM value is delivered to the program as a data area prefixed by a two-byte binary length field. The maximum length of the PARM string is **100 characters**.

In COBOL, there are two standard ways to receive this parameter:

**Method 1: ACCEPT FROM JCL (non-standard but widely supported)**

Some compilers and environments support:

```cobol
WORKING-STORAGE SECTION.
01  WS-PARM-DATA          PIC X(100).

PROCEDURE DIVISION.
    ACCEPT WS-PARM-DATA FROM JCL-PARM.
```

**Method 2: LINKAGE SECTION (standard and portable)**

The standard, portable approach uses the LINKAGE SECTION:

```cobol
LINKAGE SECTION.
01  LS-PARM-DATA.
    05  LS-PARM-LENGTH    PIC S9(04) COMP.
    05  LS-PARM-STRING    PIC X(100).

PROCEDURE DIVISION USING LS-PARM-DATA.
    IF LS-PARM-LENGTH > 0
        PERFORM PROCESS-PARAMETERS
    ELSE
        PERFORM USE-DEFAULTS
    END-IF.
```

The `PROCEDURE DIVISION USING` clause tells the runtime to map the incoming PARM area to `LS-PARM-DATA`. The first two bytes (`LS-PARM-LENGTH`) contain the binary length of the actual PARM string. If no PARM is coded on the EXEC statement, the length field is zero.

**Parsing the PARM string:**

Since the PARM is a flat character string, programs commonly use delimiter-based parsing:

```cobol
WORKING-STORAGE SECTION.
01  WS-PARM-VALUES.
    05  WS-RUN-DATE        PIC X(08).
    05  WS-RUN-MODE        PIC X(01).

PROCEDURE DIVISION USING LS-PARM-DATA.
    IF LS-PARM-LENGTH > 0
        UNSTRING LS-PARM-STRING
            DELIMITED BY ','
            INTO WS-RUN-DATE
                 WS-RUN-MODE
        END-UNSTRING
    END-IF.
```

### RETURN-CODE and the COND Parameter

When a COBOL program finishes, the value in the special register `RETURN-CODE` is passed back to JCL as the **condition code** (also called the **completion code** or **step return code**). This is a numeric value from 0 to 4095.

```cobol
PROCEDURE DIVISION.
    PERFORM MAIN-PROCESS
    IF WS-ERROR-FOUND
        MOVE 16 TO RETURN-CODE
    ELSE IF WS-WARNING-FOUND
        MOVE 4 TO RETURN-CODE
    ELSE
        MOVE 0 TO RETURN-CODE
    END-IF
    STOP RUN.
```

See [error_handling.md](error_handling.md) for the complete RETURN-CODE conventional values reference (0=success, 4=warning, 8=error, 12=severe, 16=critical).

The JCL COND parameter on the EXEC statement tests these return codes to decide whether subsequent steps should execute:

```jcl
//STEP02   EXEC PGM=RPTPRINT,COND=(8,LT)
```

The COND parameter syntax is `COND=(code,operator)` where the test is: **if the condition is TRUE, SKIP this step**. The operator compares the specified code against the return code of preceding steps:

- `COND=(8,LT)` means: if 8 is Less Than any previous step's return code, skip this step. In other words, skip if any previous step returned more than 8.
- `COND=(0,NE)` means: if 0 is Not Equal to any previous step's return code, skip this step. In other words, skip unless all previous steps returned 0.
- `COND=(4,LT,STEP01)` means: if 4 is Less Than STEP01's return code, skip this step.

Available operators: `GT`, `GE`, `EQ`, `LT`, `LE`, `NE`.

Multiple conditions can be coded:

```jcl
//STEP03   EXEC PGM=CLEANUP,COND=((8,LT,STEP01),(4,LT,STEP02))
```

This step is skipped if EITHER condition is true.

### Condition Code Handling

Modern JCL (since MVS/ESA) also supports IF/THEN/ELSE/ENDIF constructs, which are much more readable than COND:

```jcl
//         IF (STEP01.RC = 0) THEN
//STEP02   EXEC PGM=NEXTSTEP
//OUTFILE  DD DSN=PROD.OUTPUT.FILE,DISP=(NEW,CATLG,DELETE),
//            SPACE=(CYL,(10,5),RLSE)
//         ELSE
//STEP02E  EXEC PGM=ERRPROC
//         ENDIF
```

Key points about IF/THEN/ELSE:

- `RC` refers to the return code. `STEP01.RC` is step STEP01's return code.
- `ABEND` is true if the step abended. `STEP01.ABEND` tests for this.
- `ABENDCC` gives the specific abend code. `STEP01.ABENDCC = S0C7` tests for a data exception abend.
- `RUN` is true if the step actually executed (was not skipped or flushed).
- Boolean operators `AND`, `OR`, and `NOT` are supported.

```jcl
//         IF (STEP01.RC <= 4 AND STEP02.RC = 0) THEN
//STEP03   EXEC PGM=FINALRPT
//         ENDIF
```

**COND on the JOB statement:**

A COND parameter on the JOB card applies to all steps:

```jcl
//MYJOB    JOB (ACCT),'BATCH RUN',COND=(16,LT)
```

This cancels the entire job if any step returns a code greater than 16.

### SYSIN and SYSOUT

**SYSIN** is the conventional DD name for standard input. COBOL programs can read from SYSIN using `ACCEPT` statements:

```cobol
ACCEPT WS-INPUT-DATA FROM SYSIN.
```

or by defining a file assigned to SYSIN:

```cobol
SELECT CONTROL-FILE ASSIGN TO SYSIN.
```

In JCL, SYSIN is often used for inline data:

```jcl
//SYSIN    DD *
PARAM1=VALUE1
PARAM2=VALUE2
/*
```

The `DD *` indicates inline data (card-image input). `DD DATA` is the alternative that allows JCL-like statements (`//`) within the data.

**SYSOUT** is the conventional DD name for printed output. Programs direct report lines and messages to SYSOUT:

```cobol
SELECT REPORT-FILE ASSIGN TO SYSOUT.
```

In JCL:

```jcl
//SYSOUT   DD SYSOUT=*
```

The `SYSOUT=*` routes output to the default output class defined on the JOB card. A specific class can be coded:

```jcl
//SYSOUT   DD SYSOUT=A
```

Other system DD names commonly seen alongside COBOL programs:

- `SYSPRINT` -- compiler listing output during compilation
- `SYSUT1` through `SYSUT5` -- work files used by utilities
- `SYSABOUT` / `SYSDBOUT` -- debug output
- `CEEDUMP` -- Language Environment dump output on abend
- `SYSUDUMP` / `SYSABEND` / `SYSMDUMP` -- system dump DD names

### STEPLIB and JOBLIB

These DD names tell the system where to find the program load module.

**STEPLIB** applies to a single step:

```jcl
//STEP01   EXEC PGM=CUSTUPDT
//STEPLIB  DD DSN=PROD.LOADLIB,DISP=SHR
//         DD DSN=PROD.COMMON.LOADLIB,DISP=SHR
```

**JOBLIB** applies to all steps in a job (coded before the first EXEC):

```jcl
//MYJOB    JOB (ACCT),'DAILY BATCH'
//JOBLIB   DD DSN=PROD.LOADLIB,DISP=SHR
//STEP01   EXEC PGM=CUSTUPDT
```

Rules:

- If a STEPLIB is present for a step, JOBLIB is ignored for that step.
- Both can be concatenated (multiple DD statements) to search multiple libraries.
- The search order within a concatenation is top to bottom.
- If neither STEPLIB nor JOBLIB is present, only the system link list and LPA are searched.
- STEPLIB and JOBLIB must point to PDSEs or PDSs containing load modules.

### Concatenated Datasets

Multiple datasets can be concatenated under a single DD name by coding additional DD statements without a name:

```jcl
//INPUT    DD DSN=PROD.DAILY.FILE1,DISP=SHR
//         DD DSN=PROD.DAILY.FILE2,DISP=SHR
//         DD DSN=PROD.DAILY.FILE3,DISP=SHR
```

To the COBOL program, this looks like a single sequential file. When reading reaches the end of `FILE1`, the system automatically continues with `FILE2`, then `FILE3`. The program sees one continuous stream of records.

Important rules for concatenation:

- The **largest LRECL** and **largest BLKSIZE** among the concatenated datasets should be specified on the first DD or handled by the system. If the first DD has a smaller BLKSIZE than subsequent ones, an S001 abend can result.
- All datasets in a concatenation should have the same RECFM. Mixing fixed and variable records produces unpredictable results.
- For load libraries (STEPLIB/JOBLIB), concatenation defines the search order -- first match wins.
- The maximum number of datasets in a concatenation is 255.
- You cannot concatenate partitioned datasets (PDSs) with sequential datasets.

### GDG Basics for COBOL Programs

A **Generation Data Group (GDG)** is a collection of historically related sequential datasets managed under a single catalog entry. GDGs are heavily used in batch COBOL environments for maintaining rolling history.

The GDG base is a catalog entry:

```
PROD.CUSTOMER.BACKUP
```

Individual generations are referenced with relative or absolute notation:

| Reference | Meaning |
|-----------|---------|
| `PROD.CUSTOMER.BACKUP(0)` | Current (most recent) generation |
| `PROD.CUSTOMER.BACKUP(+1)` | New generation (to be created in this job) |
| `PROD.CUSTOMER.BACKUP(-1)` | Previous generation (one before current) |
| `PROD.CUSTOMER.BACKUP(-2)` | Two generations back |

In JCL:

```jcl
//* Read the current generation
//INFILE   DD DSN=PROD.CUSTOMER.BACKUP(0),DISP=SHR
//*
//* Create a new generation
//OUTFILE  DD DSN=PROD.CUSTOMER.BACKUP(+1),
//            DISP=(NEW,CATLG,DELETE),
//            SPACE=(CYL,(50,10),RLSE),
//            DCB=(RECFM=FB,LRECL=200,BLKSIZE=0)
```

The COBOL program is unaware that it is reading or writing a GDG -- it simply OPENs and processes the file as any sequential dataset. The GDG management is entirely a JCL/catalog concern.

Key GDG characteristics:

- The GDG base definition specifies a **LIMIT** (maximum number of generations to retain) and a **scratch/noscratch** option that controls whether old generations are deleted or merely uncataloged when the limit is exceeded.
- Relative references (`+1`, `0`, `-1`) are resolved at **allocation time** (job start), not at OPEN time. This means that within a single job, `(0)` always refers to the generation that was current when the job began, even if an earlier step created `(+1)`.
- A GDG generation created with `(+1)` is not fully cataloged until the step completes successfully. If the step abends, the disposition `DELETE` on the DD ensures it is removed.
- Multiple generations can be created in a single job: `(+1)`, `(+2)`, etc.
- All generations in a GDG can be referenced as a group for concatenated reading by specifying the base name without parentheses -- though this is typically done in utility steps, not COBOL programs.

## Syntax & Examples

### Complete Job with COBOL Program Execution

```jcl
//BATJOB01 JOB (ACCTG,123),'CUSTOMER UPDATE',
//             CLASS=A,MSGCLASS=X,MSGLEVEL=(1,1),
//             NOTIFY=&SYSUID
//*
//JOBLIB   DD DSN=PROD.COBOL.LOADLIB,DISP=SHR
//*
//*---------------------------------------------------
//* STEP 1: RUN THE CUSTOMER UPDATE PROGRAM
//*---------------------------------------------------
//STEP01   EXEC PGM=CUSTUPDT,PARM='20260101,F'
//*
//CUSTMAST DD DSN=PROD.CUSTOMER.MASTER,DISP=SHR
//TXNFILE  DD DSN=PROD.DAILY.TRANSACTIONS,DISP=SHR
//UPDFILE  DD DSN=PROD.CUSTOMER.MASTER.NEW,
//            DISP=(NEW,CATLG,DELETE),
//            SPACE=(CYL,(100,20),RLSE),
//            DCB=(RECFM=FB,LRECL=200,BLKSIZE=0)
//ERRFILE  DD DSN=PROD.CUSTUPDT.ERRORS,
//            DISP=(NEW,CATLG,DELETE),
//            SPACE=(TRK,(5,5),RLSE),
//            DCB=(RECFM=FB,LRECL=200,BLKSIZE=0)
//SYSOUT   DD SYSOUT=*
//CEEDUMP  DD SYSOUT=*
//*
//*---------------------------------------------------
//* STEP 2: PRINT REPORT (ONLY IF STEP01 RC <= 4)
//*---------------------------------------------------
//         IF (STEP01.RC <= 4) THEN
//STEP02   EXEC PGM=CUSTRPT
//INFILE   DD DSN=PROD.CUSTOMER.MASTER.NEW,DISP=SHR
//REPORT   DD SYSOUT=*
//         ENDIF
```

### Corresponding COBOL Program Structure (CUSTUPDT)

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. CUSTUPDT.
      *------------------------------------------------------
      * CUSTOMER MASTER UPDATE PROGRAM
      * RECEIVES PARM: RUN-DATE(8),MODE(1)
      *------------------------------------------------------
       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT CUSTOMER-MASTER
               ASSIGN TO CUSTMAST
               ORGANIZATION IS SEQUENTIAL
               ACCESS MODE IS SEQUENTIAL
               FILE STATUS IS WS-CUST-FS.
           SELECT TRANSACTION-FILE
               ASSIGN TO TXNFILE
               ORGANIZATION IS SEQUENTIAL
               FILE STATUS IS WS-TXN-FS.
           SELECT UPDATE-OUTPUT
               ASSIGN TO UPDFILE
               ORGANIZATION IS SEQUENTIAL
               FILE STATUS IS WS-UPD-FS.
           SELECT ERROR-FILE
               ASSIGN TO ERRFILE
               ORGANIZATION IS SEQUENTIAL
               FILE STATUS IS WS-ERR-FS.

       DATA DIVISION.
       FILE SECTION.
       FD  CUSTOMER-MASTER
           BLOCK CONTAINS 0 RECORDS
           RECORD CONTAINS 200 CHARACTERS.
       01  CUST-MASTER-RECORD        PIC X(200).

       FD  TRANSACTION-FILE
           BLOCK CONTAINS 0 RECORDS
           RECORD CONTAINS 200 CHARACTERS.
       01  TRANSACTION-RECORD        PIC X(200).

       FD  UPDATE-OUTPUT
           BLOCK CONTAINS 0 RECORDS
           RECORD CONTAINS 200 CHARACTERS.
       01  UPDATE-RECORD             PIC X(200).

       FD  ERROR-FILE
           BLOCK CONTAINS 0 RECORDS
           RECORD CONTAINS 200 CHARACTERS.
       01  ERROR-RECORD              PIC X(200).

       WORKING-STORAGE SECTION.
       01  WS-FILE-STATUSES.
           05  WS-CUST-FS           PIC X(02).
           05  WS-TXN-FS            PIC X(02).
           05  WS-UPD-FS            PIC X(02).
           05  WS-ERR-FS            PIC X(02).

       01  WS-PARM-VALUES.
           05  WS-RUN-DATE          PIC X(08).
           05  WS-RUN-MODE          PIC X(01).

       01  WS-FLAGS.
           05  WS-ERROR-FOUND       PIC X(01) VALUE 'N'.
               88  ERROR-FOUND      VALUE 'Y'.
               88  NO-ERRORS        VALUE 'N'.
           05  WS-WARNING-FOUND     PIC X(01) VALUE 'N'.
               88  WARNING-FOUND    VALUE 'Y'.

       01  WS-COUNTERS.
           05  WS-READ-COUNT        PIC 9(07) VALUE 0.
           05  WS-UPDATE-COUNT      PIC 9(07) VALUE 0.
           05  WS-ERROR-COUNT       PIC 9(07) VALUE 0.

       LINKAGE SECTION.
       01  LS-PARM-DATA.
           05  LS-PARM-LENGTH       PIC S9(04) COMP.
           05  LS-PARM-STRING       PIC X(100).

       PROCEDURE DIVISION USING LS-PARM-DATA.
       0000-MAIN.
           PERFORM 1000-INITIALIZE
           PERFORM 2000-PROCESS
               UNTIL WS-TXN-FS NOT = '00'
           PERFORM 9000-TERMINATE
           STOP RUN.

       1000-INITIALIZE.
           IF LS-PARM-LENGTH > 0
               UNSTRING LS-PARM-STRING
                   DELIMITED BY ','
                   INTO WS-RUN-DATE
                        WS-RUN-MODE
               END-UNSTRING
           END-IF
           OPEN INPUT  CUSTOMER-MASTER
                        TRANSACTION-FILE
                OUTPUT UPDATE-OUTPUT
                        ERROR-FILE
           IF WS-CUST-FS NOT = '00'
               DISPLAY 'CUSTMAST OPEN FAILED: ' WS-CUST-FS
               MOVE 16 TO RETURN-CODE
               STOP RUN
           END-IF
           READ TRANSACTION-FILE
               AT END SET ERROR-FOUND TO TRUE
           END-READ.

       2000-PROCESS.
           ADD 1 TO WS-READ-COUNT
           PERFORM 2100-UPDATE-MASTER
           READ TRANSACTION-FILE
               AT END MOVE '10' TO WS-TXN-FS
           END-READ.

       2100-UPDATE-MASTER.
           CONTINUE.

       9000-TERMINATE.
           CLOSE CUSTOMER-MASTER
                 TRANSACTION-FILE
                 UPDATE-OUTPUT
                 ERROR-FILE
           DISPLAY 'RECORDS READ:    ' WS-READ-COUNT
           DISPLAY 'RECORDS UPDATED: ' WS-UPDATE-COUNT
           DISPLAY 'ERRORS:          ' WS-ERROR-COUNT
           EVALUATE TRUE
               WHEN ERROR-FOUND
                   MOVE 16 TO RETURN-CODE
               WHEN WARNING-FOUND
                   MOVE 4  TO RETURN-CODE
               WHEN OTHER
                   MOVE 0  TO RETURN-CODE
           END-EVALUATE.
```

### Reading from SYSIN with Inline Data

```jcl
//STEP01   EXEC PGM=CTLREAD
//SYSIN    DD *
REGION=EAST
DATE=20260101
MODE=FULL
/*
//SYSOUT   DD SYSOUT=*
```

```cobol
       WORKING-STORAGE SECTION.
       01  WS-SYSIN-RECORD          PIC X(80).
       01  WS-SYSIN-EOF             PIC X(01) VALUE 'N'.
           88 END-OF-SYSIN          VALUE 'Y'.

       PROCEDURE DIVISION.
           OPEN INPUT SYSIN-FILE
           PERFORM UNTIL END-OF-SYSIN
               READ SYSIN-FILE INTO WS-SYSIN-RECORD
                   AT END SET END-OF-SYSIN TO TRUE
                   NOT AT END
                       PERFORM PARSE-CONTROL-RECORD
               END-READ
           END-PERFORM
           CLOSE SYSIN-FILE.
```

### GDG Processing Example

```jcl
//*---------------------------------------------------
//* READ YESTERDAY'S BACKUP, WRITE TODAY'S BACKUP
//*---------------------------------------------------
//STEP01   EXEC PGM=GDGPROC
//INFILE   DD DSN=PROD.CUSTOMER.BACKUP(0),DISP=SHR
//OUTFILE  DD DSN=PROD.CUSTOMER.BACKUP(+1),
//            DISP=(NEW,CATLG,DELETE),
//            SPACE=(CYL,(50,10),RLSE),
//            DCB=(RECFM=FB,LRECL=200,BLKSIZE=0)
//SYSOUT   DD SYSOUT=*
```

The COBOL program simply reads `INFILE` and writes `OUTFILE` -- no GDG-specific code is needed.

### Multi-Step Job with Condition Code Logic

```jcl
//DAILY    JOB (ACCTG),'DAILY CYCLE',CLASS=A,
//             MSGCLASS=X,NOTIFY=&SYSUID
//JOBLIB   DD DSN=PROD.COBOL.LOADLIB,DISP=SHR
//*
//* STEP 1: EXTRACT
//STEP01   EXEC PGM=EXTRACT
//INFILE   DD DSN=PROD.SOURCE.DATA,DISP=SHR
//OUTFILE  DD DSN=&&TEMP01,DISP=(NEW,PASS),
//            SPACE=(CYL,(10,5)),
//            DCB=(RECFM=FB,LRECL=100,BLKSIZE=0)
//SYSOUT   DD SYSOUT=*
//*
//* STEP 2: TRANSFORM (SKIP IF EXTRACT HAD ERRORS)
//         IF (STEP01.RC = 0) THEN
//STEP02   EXEC PGM=TRANSFRM
//INFILE   DD DSN=&&TEMP01,DISP=(OLD,DELETE)
//OUTFILE  DD DSN=&&TEMP02,DISP=(NEW,PASS),
//            SPACE=(CYL,(10,5)),
//            DCB=(RECFM=FB,LRECL=150,BLKSIZE=0)
//SYSOUT   DD SYSOUT=*
//         ENDIF
//*
//* STEP 3: LOAD (SKIP IF EITHER PREV STEP FAILED)
//         IF (STEP01.RC = 0 AND STEP02.RC = 0) THEN
//STEP03   EXEC PGM=LOADPGM
//INFILE   DD DSN=&&TEMP02,DISP=(OLD,DELETE)
//MASTFILE DD DSN=PROD.MASTER.FILE,DISP=OLD
//BACKUP   DD DSN=PROD.MASTER.BACKUP(+1),
//            DISP=(NEW,CATLG,DELETE),
//            SPACE=(CYL,(50,10),RLSE),
//            DCB=(RECFM=FB,LRECL=150,BLKSIZE=0)
//SYSOUT   DD SYSOUT=*
//         ENDIF
```

This pattern uses temporary datasets (`&&TEMP01`, `&&TEMP02`) passed between steps and IF/THEN/ENDIF constructs to ensure steps only run when predecessors succeed.

## Common Patterns

### Pattern 1: Standard Batch Update with Backup
The most common batch pattern: read a master file and transaction file, produce an updated master and a backup copy. The JCL creates a GDG generation for the new master and uses DISP=OLD on the existing master to ensure exclusive access.

### Pattern 2: Multi-Program Pipeline
A job runs several COBOL programs in sequence, each reading the output of the previous step. Temporary datasets (`&&name`) are used for intermediate files, with `DISP=(NEW,PASS)` in the creating step and `DISP=(OLD,DELETE)` in the consuming step. IF/THEN/ENDIF gates each step on the success of its predecessors.

### Pattern 3: Parameter-Driven Processing
A single COBOL program is reused for different processing modes by varying the PARM value. Scheduling tools (CA-7, TWS, Control-M) pass date values and mode flags via the PARM field. The program parses the PARM at initialization and branches accordingly.

### Pattern 4: IDCAMS Utility Before COBOL Step
A REPRO or DELETE/DEFINE step runs before the COBOL program to prepare VSAM files. The COBOL step's execution is conditioned on the utility step's return code:

```jcl
//STEP01   EXEC PGM=IDCAMS
//SYSPRINT DD SYSOUT=*
//SYSIN    DD *
 DELETE PROD.VSAM.FILE CLUSTER
 SET MAXCC = 0
 DEFINE CLUSTER (NAME(PROD.VSAM.FILE) ...)
/*
//*
//         IF (STEP01.RC = 0) THEN
//STEP02   EXEC PGM=VSAMLOAD
//VSAMOUT  DD DSN=PROD.VSAM.FILE,DISP=SHR
//         ENDIF
```

### Pattern 5: Concatenated Input from Multiple Sources
Multiple regional files are concatenated under a single DD name so the COBOL program reads them as one continuous file. No changes to the COBOL source are needed when regions are added or removed -- only the JCL changes.

### Pattern 6: Conditional Report Generation
A report program runs only if the main processing step completed without errors. The report reads the output of the processing step:

```jcl
//         IF (STEP01.RC <= 4) THEN
//STEP02   EXEC PGM=RPTPROG
//INFILE   DD DSN=PROD.PROCESS.OUTPUT,DISP=SHR
//REPORT   DD SYSOUT=*,DCB=(RECFM=FBA,LRECL=133)
//         ENDIF
```

The LRECL of 133 with RECFM=FBA is a classic print file convention: 1 byte for ASA carriage control plus 132 bytes of print data.

## Gotchas

- **ASSIGN TO name mismatch:** If the DD name in JCL does not exactly match the name derived from the COBOL ASSIGN clause, the OPEN statement fails. This is often caused by the ASSIGN TO prefix rules -- `ASSIGN TO S-MYFILE` uses DD name `MYFILE`, not `S-MYFILE`. The exact behavior depends on compiler options (ASSIGN(EXTERNAL) vs ASSIGN(DYNAMIC)).

- **PARM length limit of 100 characters:** The JCL PARM field is limited to 100 characters. Programs that need more input should use SYSIN or a control file instead. Exceeding this limit produces a JCL error at submission time, not at runtime.

- **COND parameter logic is inverted:** The COND parameter specifies when to SKIP a step, not when to run it. `COND=(0,NE)` means "skip if 0 is not equal to any prior return code" -- i.e., skip if any prior step returned nonzero. This backward logic is the single most common source of JCL errors. Prefer IF/THEN/ELSE constructs for clarity.

- **BLKSIZE mismatch in concatenation:** If the first dataset in a concatenation has a smaller block size than subsequent datasets, the system allocates buffers based on the first dataset's block size, causing an S001-4 abend when it encounters a larger block. Always code `BLKSIZE=0` on the first DD or ensure the first dataset has the largest block size.

- **RETURN-CODE is not preserved across CALL statements:** If a COBOL program CALLs a subprogram, and the subprogram sets RETURN-CODE, the calling program's RETURN-CODE is also affected. Always explicitly set RETURN-CODE before STOP RUN or GOBACK, not before a CALL.

- **GDG relative references resolve at job start:** `(0)` and `(+1)` are resolved when the job is allocated, not when the step runs. If STEP01 creates `(+1)` and STEP02 references `(0)` expecting to get what STEP01 created, it will instead get the generation that was current before the job started. STEP02 must reference `(+1)` to read what STEP01 wrote.

- **DISP=SHR vs DISP=OLD on input files:** Using `DISP=OLD` on an input file obtains exclusive access, preventing other jobs from reading it. For input files that do not need exclusive access, always use `DISP=SHR`. Using `DISP=OLD` unnecessarily creates batch window bottlenecks.

- **Missing CEEDUMP DD:** If a COBOL program abends under Language Environment and no `CEEDUMP DD SYSOUT=*` is present, the LE diagnostic dump is suppressed. Always include CEEDUMP in production JCL for diagnosability.

- **STEPLIB overrides JOBLIB entirely:** When a STEPLIB is present for a step, the JOBLIB is completely ignored for that step, even if the STEPLIB does not contain the needed module. This leads to S806 abends (module not found) if the STEPLIB does not include all necessary libraries.

- **FILE STATUS not checked after OPEN:** If the COBOL program does not check FILE STATUS after OPEN and the DD is missing, the program may abend with a system abend code (S013, S001) rather than handling the error gracefully. Always check FILE STATUS after every I/O operation.

- **RECFM mismatch between JCL and program:** If JCL specifies `RECFM=VB` (variable blocked) but the COBOL FD declares a fixed-length record, the runtime reads the variable-length record including the 4-byte RDW (Record Descriptor Word) into the fixed-length area, corrupting the data with no compile-time or obvious runtime error.

- **Temporary dataset names and restartability:** Temporary datasets (`&&name`) are lost if the job fails and must be restarted. For restartable jobs, use permanent dataset names with cleanup steps rather than temporary datasets.

## Related Topics

- **file_handling.md** -- Detailed coverage of COBOL file I/O operations (OPEN, CLOSE, READ, WRITE, REWRITE), file organizations (sequential, indexed, relative), and FILE STATUS codes. The JCL DD statement allocates the dataset, but the COBOL file handling logic controls how records are read and written.

- **batch_patterns.md** -- Common batch processing patterns such as sequential file update, master-transaction matching, and control break reporting. These patterns assume a JCL wrapper that provides the correct DD allocations and step sequencing.

- **error_handling.md** -- COBOL-level error handling techniques including FILE STATUS checking, USE AFTER EXCEPTION declaratives, and RETURN-CODE setting. The condition codes set in COBOL programs drive JCL-level condition code handling covered in this file.

- **cobol_structure.md** -- The four COBOL divisions and their purposes. The ENVIRONMENT DIVISION's FILE-CONTROL paragraph and the DATA DIVISION's FILE SECTION are the COBOL-side declarations that must align with JCL DD statements as described in this file.
