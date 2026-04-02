# DB2 Embedded SQL

## Description
This file covers the use of DB2 static embedded SQL within COBOL programs,
including EXEC SQL syntax, declaring and using host variables, DCLGEN,
interpreting return codes via the SQLCA, indicator variables, singleton SELECT,
cursors, INSERT/UPDATE/DELETE, the WHENEVER directive, and COMMIT/ROLLBACK.
Reference this file whenever a COBOL program interacts with a DB2 relational
database using static SQL. For dynamic SQL (EXECUTE IMMEDIATE, PREPARE/EXECUTE,
SQLDA), see db2_dynamic_sql.md.

## Table of Contents
- [Core Concepts](#core-concepts)
  - [EXEC SQL / END-EXEC Delimiters](#exec-sql--end-exec-delimiters)
  - [The DB2 Precompiler](#the-db2-precompiler)
  - [Host Variables](#host-variables)
  - [DCLGEN (Declarations Generator)](#dclgen-declarations-generator)
  - [SQLCA and SQLCODE](#sqlca-and-sqlcode)
  - [Indicator Variables](#indicator-variables)
  - [SQL INCLUDE](#sql-include)
- [Syntax & Examples](#syntax--examples)
  - [Singleton SELECT (SELECT INTO)](#singleton-select-select-into)
  - [Cursors: DECLARE, OPEN, FETCH, CLOSE](#cursors-declare-open-fetch-close)
  - [INSERT, UPDATE, DELETE](#insert-update-delete)
  - [WHENEVER Statement](#whenever-statement)
  - [COMMIT and ROLLBACK](#commit-and-rollback)
- [Common Patterns](#common-patterns)
- [Gotchas](#gotchas)
- [Related Topics](#related-topics)

## Core Concepts

### EXEC SQL / END-EXEC Delimiters

Every SQL statement embedded in a COBOL program must be enclosed between the
`EXEC SQL` and `END-EXEC` delimiters. The DB2 precompiler recognizes these
markers and replaces the enclosed SQL with COBOL CALL statements to the DB2
runtime interface.

```cobol
       EXEC SQL
           SELECT EMPLOYEE_NAME
             INTO :WS-EMP-NAME
             FROM EMPLOYEE
            WHERE EMPLOYEE_ID = :WS-EMP-ID
       END-EXEC
```

Key rules:
- `EXEC SQL` must begin in Area B (columns 12-72) just like any COBOL statement.
- Everything between `EXEC SQL` and `END-EXEC` follows SQL syntax, not COBOL
  syntax (no periods, no paragraph names).
- A period after `END-EXEC` is required only where COBOL syntax demands one
  (e.g., the last statement in a paragraph). It is common practice to place a
  period after `END-EXEC` but it is not part of the SQL itself.
- Comments inside the SQL block use SQL comment style (`--`), not COBOL (`*`).
- Only one SQL statement is permitted per `EXEC SQL ... END-EXEC` block.

### The DB2 Precompiler

Before a COBOL-DB2 program is compiled by the COBOL compiler, it must first pass
through the DB2 precompiler (DSNHPC). The precompiler:

1. Extracts all SQL statements from the source.
2. Validates SQL syntax and resolves host variable references.
3. Replaces each `EXEC SQL ... END-EXEC` block with COBOL CALL statements.
4. Produces a modified COBOL source file and a DBRM (Database Request Module).
5. The DBRM is later bound into a plan or package for execution.

The typical JCL build sequence is: Precompile, Compile, Link-Edit, Bind.

### Host Variables

Host variables are ordinary COBOL data items used to pass values between the
COBOL program and DB2. They are referenced in SQL statements with a colon
prefix (`:`).

**Declaration rules:**
- Host variables must be declared in the WORKING-STORAGE SECTION or
  LINKAGE SECTION between `EXEC SQL BEGIN DECLARE SECTION` and
  `EXEC SQL END DECLARE SECTION`, although many DB2 implementations allow
  host variables declared anywhere in these sections.
- Group-level host variables (host structures) can be used to receive an entire
  row, with each elementary item mapping to a column.
- COBOL data types must be compatible with the corresponding DB2 column types.

```cobol
       WORKING-STORAGE SECTION.

       EXEC SQL BEGIN DECLARE SECTION END-EXEC.

       01  WS-EMP-ID              PIC X(06).
       01  WS-EMP-NAME            PIC X(30).
       01  WS-SALARY              PIC S9(7)V99 COMP-3.
       01  WS-HIRE-DATE           PIC X(10).
       01  WS-DEPT-CODE           PIC S9(4) COMP.

       EXEC SQL END DECLARE SECTION END-EXEC.
```

**Common DB2-to-COBOL type mappings:**

| DB2 Column Type     | COBOL Host Variable              |
|----------------------|----------------------------------|
| CHAR(n)              | PIC X(n)                         |
| VARCHAR(n)           | 01 var. 49 len PIC S9(4) COMP. 49 data PIC X(n). |
| SMALLINT             | PIC S9(4) COMP                   |
| INTEGER              | PIC S9(9) COMP                   |
| BIGINT               | PIC S9(18) COMP                  |
| DECIMAL(p,s)         | PIC S9(p-s)V9(s) COMP-3         |
| DATE                 | PIC X(10)                        |
| TIME                 | PIC X(08)                        |
| TIMESTAMP            | PIC X(26)                        |
| FLOAT                | COMP-1 (single) or COMP-2 (double) |

**VARCHAR host variables** require a special two-level structure with a 49-level
length field followed by a 49-level data field:

```cobol
       01  WS-DESCRIPTION.
           49  WS-DESCRIPTION-LEN  PIC S9(4) COMP.
           49  WS-DESCRIPTION-TEXT PIC X(200).
```

DB2 sets `WS-DESCRIPTION-LEN` to the actual length of the data returned and
reads it on input to know how many bytes to take from the text field.

### DCLGEN (Declarations Generator)

DCLGEN is an IBM utility that generates COBOL host variable declarations and an
SQL DECLARE TABLE statement from a DB2 table definition. This eliminates manual
coding and reduces type-mismatch errors.

Running DCLGEN produces a copybook containing:
1. An `EXEC SQL DECLARE table-name TABLE (...)` statement listing all columns
   and their SQL types.
2. A COBOL host variable structure (01-level with elementary items) matching
   the table columns.

```cobol
      * DCLGEN output for table EMPLOYEE
       EXEC SQL DECLARE EMPLOYEE TABLE
           ( EMPLOYEE_ID    CHAR(6)       NOT NULL,
             EMPLOYEE_NAME  VARCHAR(30)   NOT NULL,
             SALARY         DECIMAL(9,2),
             HIRE_DATE      DATE,
             DEPT_CODE      SMALLINT
           )
       END-EXEC.

       01  DCLEMPLOYEE.
           10 EMPLOYEE-ID        PIC X(06).
           10 EMPLOYEE-NAME.
              49 EMPLOYEE-NAME-LEN  PIC S9(4) COMP.
              49 EMPLOYEE-NAME-TEXT PIC X(30).
           10 SALARY             PIC S9(7)V9(2) COMP-3.
           10 HIRE-DATE          PIC X(10).
           10 DEPT-CODE          PIC S9(4) COMP.
```

Best practices:
- Always regenerate DCLGEN members when a table definition changes.
- Include DCLGEN members via `EXEC SQL INCLUDE` rather than copying them manually.
- Store DCLGEN members in a dedicated PDS library (e.g., `project.DCLGEN.COBOL`).

### SQLCA and SQLCODE

The SQL Communication Area (SQLCA) is a structure that DB2 populates after every
SQL statement. It must be included in every COBOL-DB2 program.

```cobol
       WORKING-STORAGE SECTION.
           EXEC SQL INCLUDE SQLCA END-EXEC.
```

The SQLCA contains several fields, but the most important are:

| Field        | Type           | Description                                |
|--------------|----------------|--------------------------------------------|
| SQLCODE      | PIC S9(9) COMP | Primary return code                        |
| SQLSTATE     | PIC X(5)       | Standardized 5-character state code        |
| SQLERRMC     | PIC X(70)      | Error message tokens                       |
| SQLERRD(3)   | PIC S9(9) COMP | Row count for INSERT/UPDATE/DELETE          |
| SQLWARN0-A   | PIC X(1)       | Warning flags                              |

**Most Common SQLCODE Values:**

| SQLCODE | Meaning                                                      |
|---------|--------------------------------------------------------------|
| 0       | Successful execution                                         |
| +100    | No row found (SELECT INTO) or no more rows (FETCH)          |
| +802    | Arithmetic overflow or division by zero during evaluation    |
| -104    | Illegal symbol or token in SQL statement                     |
| -180    | Invalid date/time/timestamp string                           |
| -204    | Object not defined to DB2 (table/view/alias not found)       |
| -205    | Column name not in table                                     |
| -305    | NULL value but no indicator variable specified               |
| -501    | Cursor not open on FETCH                                     |
| -502    | Cursor already open on OPEN                                  |
| -530    | Referential integrity constraint violation (foreign key)     |
| -551    | Authorization failure; no privilege to perform operation     |
| -803    | Unique index or unique constraint violation (duplicate key)  |
| -805    | DBRM or package not found in plan                            |
| -811    | SELECT INTO returned more than one row                       |
| -818    | Timestamp mismatch between plan and DBRM (rebind needed)     |
| -904    | Resource unavailable (e.g., tablespace or index locked)      |
| -911    | Deadlock or timeout; current unit of work rolled back        |
| -913    | Deadlock or timeout; resources unavailable                   |

**SQLSTATE** provides a standardized alternative to SQLCODE. The first two
characters represent the class:
- `00` = successful completion
- `01` = warning
- `02` = no data (equivalent to SQLCODE +100)
- Everything else = error

### Indicator Variables

Indicator variables handle SQL NULLs. A COBOL program cannot represent SQL NULL
in a standard data item, so a companion indicator variable (a halfword binary
field) is used.

```cobol
       01  WS-SALARY          PIC S9(7)V99 COMP-3.
       01  WS-SALARY-IND      PIC S9(4) COMP.
```

The indicator variable is placed immediately after the host variable in SQL,
separated by no comma but optionally by the keyword `INDICATOR`:

```cobol
       EXEC SQL
           SELECT SALARY
             INTO :WS-SALARY :WS-SALARY-IND
             FROM EMPLOYEE
            WHERE EMPLOYEE_ID = :WS-EMP-ID
       END-EXEC
```

Or equivalently:

```cobol
           SELECT SALARY
             INTO :WS-SALARY INDICATOR :WS-SALARY-IND
```

**Indicator variable values after a FETCH or SELECT:**

| Value | Meaning                                                       |
|-------|---------------------------------------------------------------|
| 0     | Column value is not NULL; host variable contains valid data   |
| -1    | Column value is NULL; host variable content is undefined      |
| -2    | Column value is NULL as a result of a conversion error        |
| > 0   | Column value was truncated; indicator holds original length   |

**Using indicators on INSERT/UPDATE:** Set the indicator to `-1` to insert a
NULL, or to `0` to insert the host variable value.

```cobol
       MOVE -1 TO WS-SALARY-IND
       EXEC SQL
           UPDATE EMPLOYEE
              SET SALARY = :WS-SALARY :WS-SALARY-IND
            WHERE EMPLOYEE_ID = :WS-EMP-ID
       END-EXEC
```

### SQL INCLUDE

`EXEC SQL INCLUDE` directs the DB2 precompiler to copy source code into the
program at precompile time. This is used instead of the COBOL `COPY` verb for
DB2-related copybooks because the COBOL compiler runs after the precompiler.

```cobol
       EXEC SQL INCLUDE SQLCA END-EXEC.
       EXEC SQL INCLUDE DCLEMPLOYEE END-EXEC.
```

Key differences from COBOL COPY:
- `EXEC SQL INCLUDE` is processed by the DB2 precompiler; `COPY` is processed by
  the COBOL compiler.
- The SQLCA and DCLGEN members must be included via `EXEC SQL INCLUDE` so the
  precompiler can resolve references to SQLCODE, host variables, etc.
- The precompiler searches the libraries specified in the SYSLIB DD statement.

## Syntax & Examples

### Singleton SELECT (SELECT INTO)

A singleton SELECT retrieves exactly one row directly into host variables. If
the query returns zero rows, SQLCODE is set to +100. If it returns more than
one row, SQLCODE is set to -811.

```cobol
       EXEC SQL
           SELECT EMPLOYEE_NAME, SALARY, HIRE_DATE
             INTO :WS-EMP-NAME,
                  :WS-SALARY :WS-SALARY-IND,
                  :WS-HIRE-DATE
             FROM EMPLOYEE
            WHERE EMPLOYEE_ID = :WS-EMP-ID
       END-EXEC

       EVALUATE SQLCODE
           WHEN 0
               PERFORM PROCESS-EMPLOYEE
           WHEN +100
               DISPLAY 'EMPLOYEE NOT FOUND'
           WHEN OTHER
               PERFORM SQL-ERROR-HANDLER
       END-EVALUATE
```

### Cursors: DECLARE, OPEN, FETCH, CLOSE

Cursors process multi-row result sets one row at a time.

**DECLARE CURSOR** defines the query. This is a declarative statement; it does
not execute at runtime. It must appear before any references to the cursor in
the source, though physically it is often placed in WORKING-STORAGE or at the
beginning of the PROCEDURE DIVISION.

```cobol
       EXEC SQL
           DECLARE EMP_CURSOR CURSOR FOR
               SELECT EMPLOYEE_ID, EMPLOYEE_NAME, SALARY
                 FROM EMPLOYEE
                WHERE DEPT_CODE = :WS-DEPT-CODE
                ORDER BY EMPLOYEE_NAME
       END-EXEC
```

**WITH HOLD** keeps the cursor open across COMMIT points:

```cobol
       EXEC SQL
           DECLARE EMP_CURSOR CURSOR WITH HOLD FOR
               SELECT EMPLOYEE_ID, EMPLOYEE_NAME, SALARY
                 FROM EMPLOYEE
                WHERE DEPT_CODE = :WS-DEPT-CODE
       END-EXEC
```

**FOR UPDATE OF** declares intent to update specific columns through the cursor:

```cobol
       EXEC SQL
           DECLARE EMP_CURSOR CURSOR FOR
               SELECT EMPLOYEE_ID, EMPLOYEE_NAME, SALARY
                 FROM EMPLOYEE
                WHERE DEPT_CODE = :WS-DEPT-CODE
                FOR UPDATE OF SALARY
       END-EXEC
```

**OPEN** executes the query and positions the cursor before the first row:

```cobol
       EXEC SQL
           OPEN EMP_CURSOR
       END-EXEC

       IF SQLCODE NOT = 0
           PERFORM SQL-ERROR-HANDLER
       END-IF
```

**FETCH** retrieves the next row and advances the cursor:

```cobol
       EXEC SQL
           FETCH EMP_CURSOR
             INTO :WS-EMP-ID,
                  :WS-EMP-NAME,
                  :WS-SALARY :WS-SALARY-IND
       END-EXEC
```

**CLOSE** releases the cursor resources:

```cobol
       EXEC SQL
           CLOSE EMP_CURSOR
       END-EXEC
```

**Complete cursor processing loop:**

```cobol
       PERFORM OPEN-CURSOR
       PERFORM FETCH-ROW
       PERFORM PROCESS-ROWS UNTIL SQLCODE = +100
       PERFORM CLOSE-CURSOR
       .

       OPEN-CURSOR.
           EXEC SQL OPEN EMP_CURSOR END-EXEC
           IF SQLCODE NOT = 0
               PERFORM SQL-ERROR-HANDLER
           END-IF
           .

       FETCH-ROW.
           EXEC SQL
               FETCH EMP_CURSOR
                 INTO :WS-EMP-ID,
                      :WS-EMP-NAME,
                      :WS-SALARY :WS-SALARY-IND
           END-EXEC
           .

       PROCESS-ROWS.
           IF WS-SALARY-IND = -1
               MOVE 0 TO WS-SALARY
           END-IF
           PERFORM PROCESS-EMPLOYEE
           PERFORM FETCH-ROW
           .

       CLOSE-CURSOR.
           EXEC SQL CLOSE EMP_CURSOR END-EXEC
           .
```

### INSERT, UPDATE, DELETE

**INSERT a single row:**

```cobol
       EXEC SQL
           INSERT INTO EMPLOYEE
               (EMPLOYEE_ID, EMPLOYEE_NAME, SALARY, HIRE_DATE,
                DEPT_CODE)
           VALUES
               (:WS-EMP-ID, :WS-EMP-NAME,
                :WS-SALARY :WS-SALARY-IND,
                :WS-HIRE-DATE, :WS-DEPT-CODE)
       END-EXEC
```

**INSERT from a subselect:**

```cobol
       EXEC SQL
           INSERT INTO EMPLOYEE_ARCHIVE
               (EMPLOYEE_ID, EMPLOYEE_NAME, SALARY)
           SELECT EMPLOYEE_ID, EMPLOYEE_NAME, SALARY
             FROM EMPLOYEE
            WHERE DEPT_CODE = :WS-DEPT-CODE
       END-EXEC
```

After INSERT, `SQLERRD(3)` contains the number of rows inserted.

**UPDATE with search condition:**

```cobol
       EXEC SQL
           UPDATE EMPLOYEE
              SET SALARY = :WS-NEW-SALARY
            WHERE DEPT_CODE = :WS-DEPT-CODE
              AND SALARY < :WS-MIN-SALARY
       END-EXEC

       MOVE SQLERRD(3) TO WS-ROWS-UPDATED
```

**Positioned UPDATE (using cursor):**

```cobol
       EXEC SQL
           UPDATE EMPLOYEE
              SET SALARY = :WS-NEW-SALARY
            WHERE CURRENT OF EMP_CURSOR
       END-EXEC
```

**DELETE with search condition:**

```cobol
       EXEC SQL
           DELETE FROM EMPLOYEE
            WHERE EMPLOYEE_ID = :WS-EMP-ID
       END-EXEC
```

**Positioned DELETE:**

```cobol
       EXEC SQL
           DELETE FROM EMPLOYEE
            WHERE CURRENT OF EMP_CURSOR
       END-EXEC
```

After UPDATE or DELETE, check `SQLERRD(3)` for the number of rows affected
and SQLCODE +100 if no rows matched.

### WHENEVER Statement

`EXEC SQL WHENEVER` provides automatic branching after SQL errors or warnings.
It is a precompiler directive that generates an `IF SQLCODE ...` test after
every subsequent SQL statement in the source.

```cobol
       EXEC SQL
           WHENEVER SQLERROR GO TO SQL-ERROR-PARA
       END-EXEC

       EXEC SQL
           WHENEVER SQLWARNING CONTINUE
       END-EXEC

       EXEC SQL
           WHENEVER NOT FOUND GO TO NO-MORE-ROWS
       END-EXEC
```

**Condition types:**

| Condition    | Triggers when                      |
|--------------|------------------------------------|
| SQLERROR     | SQLCODE < 0                        |
| SQLWARNING   | SQLWARN0 = 'W' (any warning flag)  |
| NOT FOUND    | SQLCODE = +100                     |

**Action types:**

| Action       | Effect                                          |
|--------------|-------------------------------------------------|
| CONTINUE     | Take no automatic action; program continues     |
| GO TO label  | Branch to the specified paragraph or section     |

Important: WHENEVER is positional in the source code. It applies to all SQL
statements that appear after it in the source until another WHENEVER for the
same condition overrides it. It is not scoped to a paragraph or section.

### COMMIT and ROLLBACK

DB2 uses units of work (transactions). All changes are provisional until
explicitly committed or rolled back.

```cobol
       EXEC SQL COMMIT END-EXEC.
```

```cobol
       EXEC SQL ROLLBACK END-EXEC.
```

**SAVEPOINT** allows partial rollback within a unit of work:

```cobol
       EXEC SQL SAVEPOINT SP1 ON ROLLBACK RETAIN CURSORS END-EXEC

      * ... perform some updates ...

       EXEC SQL ROLLBACK TO SAVEPOINT SP1 END-EXEC
```

In a CICS environment, `EXEC SQL COMMIT` and `EXEC SQL ROLLBACK` must not be
used. Instead, use CICS SYNCPOINT commands, which coordinate both CICS and DB2
resources. In batch programs, explicit COMMIT/ROLLBACK is standard.

## Common Patterns

### Standard Error-Handling Paragraph

Most production COBOL-DB2 programs include a centralized error handler:

```cobol
       SQL-ERROR-HANDLER.
           EVALUATE SQLCODE
               WHEN 0
                   CONTINUE
               WHEN +100
                   SET WS-NO-DATA-FOUND TO TRUE
               WHEN -305
                   DISPLAY 'NULL INDICATOR MISSING'
                   DISPLAY 'SQLERRMC: ' SQLERRMC
               WHEN -803
                   DISPLAY 'DUPLICATE KEY VIOLATION'
               WHEN -805
                   DISPLAY 'DBRM/PACKAGE NOT FOUND - REBIND'
               WHEN -811
                   DISPLAY 'MULTIPLE ROWS FROM SELECT INTO'
               WHEN -818
                   DISPLAY 'TIMESTAMP MISMATCH - REBIND'
               WHEN -904
                   DISPLAY 'RESOURCE UNAVAILABLE'
               WHEN -911
                   DISPLAY 'DEADLOCK - ROLLING BACK'
                   EXEC SQL ROLLBACK END-EXEC
               WHEN -913
                   DISPLAY 'TIMEOUT - ROLLING BACK'
                   EXEC SQL ROLLBACK END-EXEC
               WHEN OTHER
                   DISPLAY 'UNEXPECTED SQLCODE: ' SQLCODE
                   DISPLAY 'SQLSTATE: ' SQLSTATE
                   DISPLAY 'SQLERRMC: ' SQLERRMC
           END-EVALUATE
           .
```

### Batch Commit Strategy

Long-running batch programs should commit periodically to avoid lock escalation
and to provide restart capability:

```cobol
       01  WS-COMMIT-COUNT     PIC S9(9) COMP VALUE 0.
       01  WS-COMMIT-FREQ      PIC S9(9) COMP VALUE 500.

       PROCESS-AND-COMMIT.
           PERFORM UPDATE-ROW
           ADD 1 TO WS-COMMIT-COUNT
           IF WS-COMMIT-COUNT >= WS-COMMIT-FREQ
               EXEC SQL COMMIT END-EXEC
               MOVE 0 TO WS-COMMIT-COUNT
               DISPLAY 'COMMITTED AT ' FUNCTION CURRENT-DATE
           END-IF
           .
```

When using periodic commits with a cursor, declare the cursor `WITH HOLD` to
keep it open across commit points.

### Defensive NULL Handling

Always provide indicator variables for nullable columns. A common defensive
pattern initializes indicators before each FETCH:

```cobol
       INIT-INDICATORS.
           MOVE 0 TO WS-SALARY-IND
           MOVE 0 TO WS-BONUS-IND
           MOVE 0 TO WS-COMM-IND
           .

       FETCH-AND-CHECK.
           PERFORM INIT-INDICATORS
           EXEC SQL
               FETCH EMP_CURSOR
                 INTO :WS-EMP-ID,
                      :WS-SALARY  :WS-SALARY-IND,
                      :WS-BONUS   :WS-BONUS-IND,
                      :WS-COMM    :WS-COMM-IND
           END-EXEC
           IF SQLCODE = 0
               IF WS-SALARY-IND < 0
                   MOVE 0 TO WS-SALARY
               END-IF
               IF WS-BONUS-IND < 0
                   MOVE 0 TO WS-BONUS
               END-IF
               IF WS-COMM-IND < 0
                   MOVE 0 TO WS-COMM
               END-IF
           END-IF
           .
```

### Multi-Row FETCH

For performance, DB2 supports fetching multiple rows at once into a host
variable array (OCCURS DEPENDING ON or ROWSET processing):

```cobol
       01  WS-EMP-ARRAY.
           05  WS-EMP-ROW  OCCURS 100 TIMES.
               10  WS-ARR-EMP-ID    PIC X(06).
               10  WS-ARR-EMP-NAME  PIC X(30).

       01  WS-ROWS-FETCHED  PIC S9(9) COMP.

       EXEC SQL
           DECLARE MULTI_CURSOR CURSOR WITH ROWSET POSITIONING FOR
               SELECT EMPLOYEE_ID, EMPLOYEE_NAME
                 FROM EMPLOYEE
                WHERE DEPT_CODE = :WS-DEPT-CODE
       END-EXEC

       EXEC SQL OPEN MULTI_CURSOR END-EXEC

       EXEC SQL
           FETCH NEXT ROWSET FROM MULTI_CURSOR
               FOR 100 ROWS
             INTO :WS-ARR-EMP-ID, :WS-ARR-EMP-NAME
       END-EXEC

       MOVE SQLERRD(3) TO WS-ROWS-FETCHED
```

## Gotchas

- **SQLCODE -811 from SELECT INTO:** A singleton SELECT that returns more than
  one row produces -811 and no data is placed in host variables. Always ensure
  your WHERE clause uniquely identifies a single row, or use a cursor instead.

- **Forgetting indicator variables on nullable columns (SQLCODE -305):** If a
  column can contain NULL and no indicator variable is provided, DB2 returns
  SQLCODE -305. Always pair nullable columns with indicator variables.

- **WHENEVER scope is positional, not logical:** `EXEC SQL WHENEVER SQLERROR
  GO TO ...` applies to all SQL statements that follow it in the source file,
  regardless of paragraph or section boundaries. A common bug is failing to
  reset it with `WHENEVER SQLERROR CONTINUE` before SQL statements that should
  handle errors inline, leading to unexpected branching.

- **WHENEVER NOT FOUND with cursors:** If `WHENEVER NOT FOUND GO TO ...` is
  active, the FETCH that returns +100 will branch immediately, potentially
  skipping CLOSE cursor logic. Many shops avoid WHENEVER entirely and test
  SQLCODE explicitly.

- **Cursor not closed before reopening (SQLCODE -502):** Attempting to OPEN a
  cursor that is already open produces -502. Always CLOSE a cursor before
  opening it again, or check and handle the -502 condition.

- **DBRM/plan timestamp mismatch (SQLCODE -818):** After recompiling, if you
  forget to rebind the DBRM into the plan or package, DB2 detects the timestamp
  mismatch at runtime. Always rebind after precompiling.

- **DBRM not found (SQLCODE -805):** The plan or package cannot find the DBRM
  member. This usually means the BIND step was skipped or bound to a different
  plan/collection.

- **Cursor WITH HOLD and COMMIT interaction:** Without `WITH HOLD`, all cursors
  are closed when COMMIT executes. If a batch program commits mid-cursor without
  WITH HOLD, the next FETCH fails with -501.

- **VARCHAR host variable structure:** The two-level 49-level structure is
  mandatory for VARCHAR columns. Using a simple PIC X field causes truncation
  or padding mismatches, and DB2 will not set the length portion.

- **Trailing spaces in CHAR columns:** DB2 CHAR columns are fixed-length and
  padded with spaces. Comparisons in WHERE clauses should account for this.
  VARCHAR columns do not have trailing-space padding.

- **COMMIT/ROLLBACK in CICS:** Issuing `EXEC SQL COMMIT` or `EXEC SQL ROLLBACK`
  in a CICS program can cause unpredictable results because CICS manages the
  unit of work. Use `EXEC CICS SYNCPOINT` and `EXEC CICS SYNCPOINT ROLLBACK`
  instead.

- **SQLERRD(3) after a failed statement:** After a negative SQLCODE, SQLERRD(3)
  may not contain a meaningful row count. Only rely on it when SQLCODE is 0 or
  +100.

- **DECLARE CURSOR placement:** DECLARE CURSOR is a declarative statement
  processed at precompile time, not at runtime. It must appear before the first
  OPEN for that cursor in the source, but placing it in WORKING-STORAGE or at
  the top of PROCEDURE DIVISION is common and valid.

- **Host variable alignment with DCLGEN:** If a table is altered (columns added,
  types changed), the DCLGEN member becomes stale. Using an outdated DCLGEN can
  cause SQLCODE -303 (type mismatch) or -301 (incompatible types) at runtime.

## Related Topics

- **[db2_dynamic_sql.md](db2_dynamic_sql.md)** - Dynamic SQL techniques including
  EXECUTE IMMEDIATE, PREPARE/EXECUTE, cursor-based dynamic SQL, SQLDA, parameter
  markers, and building dynamic WHERE clauses.
- **data_types.md** - COBOL PIC clauses and USAGE types that correspond to DB2
  column types; essential for defining correct host variables.
- **error_handling.md** - General COBOL error-handling techniques that complement
  SQLCODE checking and the WHENEVER directive.
- **copybooks.md** - DCLGEN members are stored and managed as copybooks; covers
  the COPY verb and library management.
- **working_storage.md** - Host variables, the SQLCA, and indicator variables
  are all declared in WORKING-STORAGE; covers declaration rules and layout.
- **cics.md** - CICS-DB2 interaction, including SYNCPOINT usage instead of
  COMMIT/ROLLBACK, and thread management for DB2 calls in online programs.
