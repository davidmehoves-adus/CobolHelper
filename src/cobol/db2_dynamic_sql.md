# DB2 Dynamic SQL

## Description
This file covers dynamic SQL techniques in COBOL-DB2 programs, where SQL
statements are constructed and executed at runtime rather than being fixed at
precompile time. It covers EXECUTE IMMEDIATE for one-time statements,
PREPARE/EXECUTE for parameterized statements, cursor-based dynamic SQL for
result sets, the SQLDA (SQL Descriptor Area) for varying-list queries, parameter
markers, and patterns for building dynamic WHERE clauses. Reference this file
when a COBOL program must construct SQL statements at runtime based on
variable criteria, user input, or configuration data.

## Table of Contents
- [Core Concepts](#core-concepts)
  - [Static vs Dynamic SQL](#static-vs-dynamic-sql)
  - [Levels of Dynamic SQL](#levels-of-dynamic-sql)
  - [Parameter Markers](#parameter-markers)
  - [SQLDA (SQL Descriptor Area)](#sqlda-sql-descriptor-area)
- [Syntax & Examples](#syntax--examples)
  - [EXECUTE IMMEDIATE](#execute-immediate)
  - [PREPARE and EXECUTE](#prepare-and-execute)
  - [Cursor-Based Dynamic SQL](#cursor-based-dynamic-sql)
  - [DESCRIBE and the SQLDA](#describe-and-the-sqlda)
  - [Building Dynamic WHERE Clauses](#building-dynamic-where-clauses)
- [Common Patterns](#common-patterns)
- [Gotchas](#gotchas)
- [Related Topics](#related-topics)

## Core Concepts

### Static vs Dynamic SQL

**Static SQL** is fully known at precompile time. The SQL text is fixed in the
source code and is bound into a package or plan. DB2 determines the access path
at BIND time.

Advantages of static SQL:
- Access path is optimized at BIND time.
- Authorization is checked at BIND time, not at runtime.
- Generally better performance for repetitive, predictable queries.
- Easier to audit and maintain.

**Dynamic SQL** is constructed at runtime as a character string. The SQL text is
not known until execution. DB2 must parse, validate, authorize, and optimize the
statement at runtime.

Dynamic SQL is appropriate when:
- The table name, column list, or WHERE clause conditions are not known until runtime.
- The program must execute DDL statements (CREATE, ALTER, DROP).
- Search criteria are optional and vary between executions.
- The program serves as a general-purpose query tool.

### Levels of Dynamic SQL

There are four levels of dynamic SQL, each with increasing complexity:

1. **Non-SELECT (EXECUTE IMMEDIATE):** For DDL or DML that does not return rows.
   Simplest form; the statement is parsed, optimized, and executed in one step.
2. **Non-SELECT (PREPARE + EXECUTE):** For DML that does not return rows but
   will be executed multiple times with different parameter values. The statement
   is parsed and optimized once (PREPARE), then executed multiple times (EXECUTE).
3. **Fixed-list SELECT (PREPARE + DECLARE + OPEN + FETCH + CLOSE):** The number
   and types of result columns are known in advance. The program declares host
   variables to receive the results.
4. **Varying-list SELECT (uses SQLDA):** The result columns are not known until
   runtime; the program inspects the SQLDA to determine column count, types,
   and lengths, then allocates storage dynamically.

### Parameter Markers

Parameter markers (`?`) are placeholders in dynamic SQL statements that
represent host variable values to be supplied at EXECUTE or OPEN time. They
serve the same role as host variables (`:variable`) in static SQL.

```sql
UPDATE EMPLOYEE SET SALARY = ? WHERE EMPLOYEE_ID = ?
```

Key rules:
- Parameter markers can appear wherever a host variable could appear in static SQL.
- They cannot replace table names, column names, or SQL keywords.
- The data types of parameter markers are inferred from context (e.g., the column
  type in a comparison or assignment).
- Using parameter markers with PREPARE/EXECUTE is more efficient than
  concatenating values into the SQL string and using EXECUTE IMMEDIATE.
- Parameter markers prevent SQL injection by separating SQL structure from data.

### SQLDA (SQL Descriptor Area)

The SQLDA is a variable-length structure that describes the columns of a dynamic
SQL result set or the parameters of a prepared statement. It is used in
varying-list dynamic SQL where the program does not know the result columns
at compile time.

The SQLDA contains:
- **SQLDABC**: Total length of the SQLDA in bytes.
- **SQLN**: Number of SQLVAR entries allocated (set by the program before DESCRIBE).
- **SQLD**: Number of columns actually described (set by DB2 after DESCRIBE).
- **SQLVAR array**: One entry per column, each containing:
  - **SQLTYPE**: Data type code and nullability indicator.
  - **SQLLEN**: Length of the column data.
  - **SQLDATA**: Pointer to the host variable that will receive the data.
  - **SQLIND**: Pointer to the indicator variable.
  - **SQLNAME**: Column name.

To use the SQLDA, the program must:
1. Allocate an SQLDA with enough SQLVAR entries.
2. PREPARE the statement.
3. DESCRIBE the prepared statement INTO the SQLDA.
4. Inspect SQLD to determine the number of columns.
5. For each column, examine SQLTYPE and SQLLEN, allocate appropriate storage,
   and set SQLDATA and SQLIND pointers.
6. OPEN the cursor and FETCH rows using the SQLDA.

## Syntax & Examples

### EXECUTE IMMEDIATE

EXECUTE IMMEDIATE parses, optimizes, and executes a dynamic SQL statement in
one step. It is used for statements that return no result set and will be
executed only once.

```cobol
       01  WS-SQL-STMT     PIC X(500).

       MOVE 'DELETE FROM TEMP_TABLE WHERE STATUS = ''X'''
           TO WS-SQL-STMT

       EXEC SQL
           EXECUTE IMMEDIATE :WS-SQL-STMT
       END-EXEC

       IF SQLCODE NOT = 0
           DISPLAY 'EXECUTE IMMEDIATE FAILED: ' SQLCODE
       END-IF
```

EXECUTE IMMEDIATE cannot be used with:
- SELECT statements (they return rows; use a cursor instead).
- Statements containing parameter markers (use PREPARE/EXECUTE instead).

### PREPARE and EXECUTE

PREPARE parses and optimizes a dynamic SQL statement once so it can be executed
multiple times efficiently. Parameter markers (`?`) serve as placeholders for
host variables supplied at EXECUTE time.

**PREPARE and EXECUTE for non-SELECT:**

```cobol
       01  WS-SQL-STMT     PIC X(500).

       MOVE 'UPDATE EMPLOYEE SET SALARY = ? WHERE EMPLOYEE_ID = ?'
           TO WS-SQL-STMT

       EXEC SQL
           PREPARE STMT1 FROM :WS-SQL-STMT
       END-EXEC

       IF SQLCODE NOT = 0
           DISPLAY 'PREPARE FAILED: ' SQLCODE
       END-IF

       EXEC SQL
           EXECUTE STMT1 USING :WS-NEW-SALARY, :WS-EMP-ID
       END-EXEC
```

The USING clause on EXECUTE supplies values for each `?` parameter marker,
matched left to right. The number of host variables in USING must equal the
number of parameter markers in the prepared statement.

A prepared statement can be executed repeatedly with different values:

```cobol
       PERFORM VARYING WS-IDX FROM 1 BY 1
               UNTIL WS-IDX > WS-UPDATE-COUNT
           MOVE WS-NEW-SAL(WS-IDX) TO WS-NEW-SALARY
           MOVE WS-EMP(WS-IDX) TO WS-EMP-ID
           EXEC SQL
               EXECUTE STMT1 USING :WS-NEW-SALARY, :WS-EMP-ID
           END-EXEC
       END-PERFORM
```

### Cursor-Based Dynamic SQL

When a dynamic SELECT must return multiple rows, combine PREPARE with a cursor.

**Fixed-list dynamic cursor (column types known at compile time):**

```cobol
       MOVE 'SELECT EMPLOYEE_ID, EMPLOYEE_NAME FROM EMPLOYEE
      -      ' WHERE DEPT_CODE = ?'
           TO WS-SQL-STMT

       EXEC SQL
           PREPARE STMT2 FROM :WS-SQL-STMT
       END-EXEC

       EXEC SQL
           DECLARE DYN_CURSOR CURSOR FOR STMT2
       END-EXEC

       EXEC SQL
           OPEN DYN_CURSOR USING :WS-DEPT-CODE
       END-EXEC

       PERFORM UNTIL SQLCODE = +100
           EXEC SQL
               FETCH DYN_CURSOR
                 INTO :WS-EMP-ID, :WS-EMP-NAME
           END-EXEC
           IF SQLCODE = 0
               PERFORM PROCESS-EMPLOYEE
           END-IF
       END-PERFORM

       EXEC SQL
           CLOSE DYN_CURSOR
       END-EXEC
```

The USING clause on OPEN supplies values for parameter markers in the prepared
statement's WHERE clause. The INTO clause on FETCH receives result columns into
host variables, just as with a static cursor.

### DESCRIBE and the SQLDA

DESCRIBE populates an SQLDA with information about a prepared statement's
result columns, enabling varying-list dynamic SQL where the program discovers
the result structure at runtime.

```cobol
       EXEC SQL
           DESCRIBE STMT2 INTO :SQLDA
       END-EXEC
```

After DESCRIBE:
- SQLD contains the number of result columns.
- If SQLD > SQLN, the SQLDA was too small; reallocate with more SQLVAR entries
  and DESCRIBE again.
- Each SQLVAR entry describes one column: its type (SQLTYPE), length (SQLLEN),
  and name (SQLNAME).

The program must then set SQLDATA and SQLIND pointers in each SQLVAR entry
to point to appropriately sized storage areas before issuing FETCH.

Varying-list dynamic SQL is complex and relatively uncommon in standard batch
COBOL programs. It is most often found in general-purpose query tools or
interactive CICS applications that allow users to select arbitrary columns.

### Building Dynamic WHERE Clauses

A common pattern constructs SQL dynamically based on user-supplied search
criteria, using parameter markers for safety and efficiency:

```cobol
       01  WS-SQL-STMT    PIC X(1000).
       01  WS-WHERE-ADDED PIC X VALUE 'N'.

       BUILD-DYNAMIC-SQL.
           MOVE 'SELECT EMPLOYEE_ID, EMPLOYEE_NAME FROM EMPLOYEE'
               TO WS-SQL-STMT

           IF WS-SEARCH-DEPT NOT = SPACES
               STRING WS-SQL-STMT DELIMITED SPACES
                      ' WHERE DEPT_CODE = ?' DELIMITED SIZE
                   INTO WS-SQL-STMT
               END-STRING
               MOVE 'Y' TO WS-WHERE-ADDED
           END-IF

           IF WS-SEARCH-STATUS NOT = SPACES
               IF WS-WHERE-ADDED = 'Y'
                   STRING WS-SQL-STMT DELIMITED SPACES
                          ' AND STATUS = ?' DELIMITED SIZE
                       INTO WS-SQL-STMT
                   END-STRING
               ELSE
                   STRING WS-SQL-STMT DELIMITED SPACES
                          ' WHERE STATUS = ?' DELIMITED SIZE
                       INTO WS-SQL-STMT
                   END-STRING
               END-IF
           END-IF

           EXEC SQL
               PREPARE DYN_STMT FROM :WS-SQL-STMT
           END-EXEC
           .
```

The corresponding OPEN or EXECUTE USING clause must supply host variables
matching only the parameter markers actually present in the constructed
statement. This requires the program to track which criteria were included
and pass the correct number of host variables.

## Common Patterns

### Reusable Prepared Statement

Prepare once at initialization, execute many times during processing:

```cobol
       1000-INITIALIZE.
           MOVE 'INSERT INTO AUDIT_LOG (TIMESTAMP, ACTION, DETAIL)
      -          VALUES (CURRENT TIMESTAMP, ?, ?)'
               TO WS-SQL-STMT
           EXEC SQL
               PREPARE AUDIT-STMT FROM :WS-SQL-STMT
           END-EXEC
           .

       2500-LOG-AUDIT.
           EXEC SQL
               EXECUTE AUDIT-STMT
                   USING :WS-ACTION-CODE, :WS-DETAIL-TEXT
           END-EXEC
           .
```

### Dynamic Table Name Selection

Since parameter markers cannot replace table names, use string concatenation
for the table name portion and parameter markers for data values:

```cobol
           STRING 'SELECT COUNT(*) FROM '
                  WS-TABLE-NAME DELIMITED SPACES
                  ' WHERE STATUS = ?'  DELIMITED SIZE
               INTO WS-SQL-STMT
           END-STRING
           EXEC SQL
               PREPARE COUNT-STMT FROM :WS-SQL-STMT
           END-EXEC
           EXEC SQL
               DECLARE COUNT-CSR CURSOR FOR COUNT-STMT
           END-EXEC
           EXEC SQL
               OPEN COUNT-CSR USING :WS-STATUS-VALUE
           END-EXEC
           EXEC SQL
               FETCH COUNT-CSR INTO :WS-ROW-COUNT
           END-EXEC
           EXEC SQL
               CLOSE COUNT-CSR
           END-EXEC
           .
```

### EXECUTE IMMEDIATE for DDL

Dynamic SQL is the only way to issue DDL (CREATE, ALTER, DROP, GRANT) from
a COBOL program:

```cobol
           STRING 'CREATE INDEX IX_'
                  WS-TABLE-NAME DELIMITED SPACES
                  ' ON '
                  WS-TABLE-NAME DELIMITED SPACES
                  ' (' WS-COLUMN-NAME DELIMITED SPACES ')'
                  DELIMITED SIZE
               INTO WS-SQL-STMT
           END-STRING
           EXEC SQL
               EXECUTE IMMEDIATE :WS-SQL-STMT
           END-EXEC
           .
```

## Gotchas

- **EXECUTE IMMEDIATE cannot handle SELECT:** Attempting to execute a SELECT
  statement via EXECUTE IMMEDIATE produces SQLCODE -518. Use PREPARE with a
  cursor for SELECT statements.

- **EXECUTE IMMEDIATE cannot use parameter markers:** Parameter markers (`?`)
  are not valid in EXECUTE IMMEDIATE. Use PREPARE/EXECUTE instead, or
  concatenate literal values into the SQL string (with proper quoting).

- **Parameter marker count mismatch:** If the number of host variables in
  EXECUTE USING or OPEN USING does not match the number of `?` markers in the
  prepared statement, DB2 returns SQLCODE -313.

- **Dynamic SQL and SQL injection:** When building SQL strings dynamically from
  user input, validate and sanitize all input. Use parameter markers (`?`) with
  PREPARE/EXECUTE instead of concatenating values directly into the SQL string.

- **Prepared statement persistence:** A prepared statement is associated with a
  thread/connection. In batch programs, it persists until the program ends, the
  statement is re-prepared, or a COMMIT occurs (unless the package is bound with
  KEEPDYNAMIC(YES)). Without KEEPDYNAMIC(YES), the prepared statement is
  invalidated at COMMIT, requiring re-preparation.

- **DESCRIBE before FETCH is required for varying-list:** If the program does not
  know the result columns at compile time, it must DESCRIBE the prepared statement
  to populate the SQLDA before FETCH. Attempting to FETCH without properly setting
  up SQLDATA pointers causes unpredictable results or abends.

- **Mixing static and dynamic SQL on the same table:** Be aware that static and
  dynamic SQL may use different access paths and different authorization models.
  Static SQL uses the authority of the plan/package owner; dynamic SQL uses the
  authority of the user running the program (unless DYNAMICRULES BIND is set).

- **Dynamic SQL performance overhead:** Each EXECUTE IMMEDIATE or PREPARE incurs
  parsing and optimization cost. For frequently executed statements, PREPARE once
  and EXECUTE multiple times rather than using EXECUTE IMMEDIATE in a loop.

- **SQLDA sizing:** If the SQLDA's SQLN is smaller than the number of result
  columns (SQLD after DESCRIBE), the SQLDA is incomplete. The program must
  reallocate a larger SQLDA and DESCRIBE again.

- **Cursor name conflicts:** Dynamic cursor names share the namespace with static
  cursor names. Ensure dynamic cursor names do not collide with any static cursor
  declarations in the same program.

## Related Topics

- **[db2_embedded_sql.md](db2_embedded_sql.md)** - Static embedded SQL
  techniques including EXEC SQL syntax, host variables, DCLGEN, SQLCA/SQLCODE,
  indicator variables, cursors, INSERT/UPDATE/DELETE, WHENEVER, and
  COMMIT/ROLLBACK.
- **data_types.md** - COBOL PIC clauses and USAGE types that correspond to DB2
  column types; essential for defining host variables used with dynamic SQL.
- **error_handling.md** - General COBOL error-handling techniques that complement
  SQLCODE checking in dynamic SQL programs.
- **string_handling.md** - STRING and UNSTRING operations used to build dynamic
  SQL statements from component parts.
