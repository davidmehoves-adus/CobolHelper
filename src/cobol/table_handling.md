# Table Handling

## Description
This file covers COBOL table (array) handling, including how to define fixed and variable-length tables with the OCCURS clause, how to access elements via subscripts and indexes, and how to search tables using SEARCH and SEARCH ALL. Reference this file when working with repeating data structures, lookup tables, accumulators, or any scenario involving the OCCURS, SEARCH, SET, or INDEXED BY clauses.

## Table of Contents
- [Core Concepts](#core-concepts)
  - [What Is a Table?](#what-is-a-table)
  - [Subscripts vs Indexes](#subscripts-vs-indexes)
  - [Fixed-Length vs Variable-Length Tables](#fixed-length-vs-variable-length-tables)
  - [Multi-Dimensional Tables](#multi-dimensional-tables)
  - [Table Keys](#table-keys)
- [Syntax & Examples](#syntax--examples)
  - [OCCURS Clause](#occurs-clause)
  - [OCCURS DEPENDING ON](#occurs-depending-on)
  - [INDEXED BY Clause](#indexed-by-clause)
  - [SET Statement](#set-statement)
  - [SEARCH (Serial Search)](#search-serial-search)
  - [SEARCH ALL (Binary Search)](#search-all-binary-search)
  - [REDEFINES with Tables](#redefines-with-tables)
  - [Multi-Dimensional Table Definition](#multi-dimensional-table-definition)
- [Common Patterns](#common-patterns)
- [Gotchas](#gotchas)
- [Related Topics](#related-topics)

## Core Concepts

### What Is a Table?

A COBOL table is an ordered, fixed-size collection of identically structured data items, analogous to an array in other languages. Tables are defined in the DATA DIVISION using the OCCURS clause. Every element in a table shares the same PICTURE, USAGE, and subordinate structure. COBOL does not support dynamically resizable arrays; the maximum size must be known at compile time.

Tables are stored contiguously in memory. A table of 100 five-byte elements occupies exactly 500 bytes. This contiguous layout is important for understanding performance and for techniques like REDEFINES overlays.

### Subscripts vs Indexes

COBOL provides two mechanisms to reference individual table elements: **subscripts** and **indexes**. Understanding the difference is critical for writing correct and efficient programs.

**Subscripts** are ordinary numeric data items or integer literals used in parentheses after a table element name. Subscripts are 1-based (the first element is element 1, not 0). A subscript can be any arithmetic expression that evaluates to a positive integer. At runtime the compiler must multiply the subscript value by the element length and add it to the base address on every reference.

**Indexes** are special internal items declared with the INDEXED BY clause. An index stores a **byte displacement** from the start of the table rather than an occurrence number. Because the displacement is pre-calculated, index-based references avoid the multiply-and-add step required for subscripts. Indexes are required for SEARCH and SEARCH ALL statements. You manipulate indexes with the SET statement, not MOVE or ADD.

Key differences at a glance:

| Aspect | Subscript | Index |
|---|---|---|
| Declared via | Normal PIC or numeric literal | INDEXED BY on the OCCURS item |
| Internal representation | Occurrence number (1-based integer) | Byte displacement from table start |
| Modification | MOVE, ADD, SUBTRACT, COMPUTE | SET, SEARCH (automatic), SET UP/DOWN BY |
| Can be used in SEARCH? | No (SEARCH requires an index) | Yes |
| Display/print directly? | Yes | No (must SET a numeric item FROM the index) |
| Arithmetic expressions allowed? | Yes | No |
| Performance | Slightly slower (runtime multiply) | Slightly faster (pre-computed offset) |

### Fixed-Length vs Variable-Length Tables

A **fixed-length table** uses a simple OCCURS clause specifying an exact number of occurrences. The compiler allocates space for exactly that many elements.

A **variable-length table** uses OCCURS ... DEPENDING ON (ODO), where the actual number of active elements is controlled by a data-name at runtime. The compiler allocates space for the maximum number of elements, but statements like SEARCH and WRITE respect the current value of the DEPENDING ON object. The ODO object (the controlling data-name) must be defined before the ODO subject (the table) in the record, and must be an integer item.

### Multi-Dimensional Tables

COBOL supports tables within tables, up to seven levels of nesting in most compilers (the COBOL standard allows at least three; IBM and Micro Focus support seven). A two-dimensional table is a table whose elements themselves contain an OCCURS clause. When referencing an element in a multi-dimensional table, you supply subscripts or indexes from the outermost dimension to the innermost, left to right.

For example, with a 12-month by 31-day structure, you reference an element as `DAILY-AMOUNT(MONTH-IDX, DAY-IDX)` where the first subscript/index identifies the month (outer) and the second identifies the day (inner).

### Table Keys

The ASCENDING KEY and DESCENDING KEY phrases on the OCCURS clause declare which fields serve as sort keys for the table. These key declarations are mandatory for SEARCH ALL (binary search) and tell the compiler which field to compare and in what order. The data in the table must actually be sorted in the declared key order before SEARCH ALL is executed; the compiler does not verify or enforce this at runtime.

Multiple keys can be declared, forming a composite key evaluated left to right (major to minor).

## Syntax & Examples

### OCCURS Clause

The OCCURS clause may appear on any data item at level 02 through 49. It cannot appear on a level 01, 66, 77, or 88 item.

```cobol
       01  EMPLOYEE-TABLE.
           05  EMPLOYEE-ENTRY OCCURS 100 TIMES.
               10  EMP-ID          PIC X(06).
               10  EMP-NAME        PIC X(30).
               10  EMP-SALARY      PIC 9(07)V99.
```

This creates a table of 100 employee entries. To reference the name of the 5th employee:

```cobol
       MOVE EMP-NAME(5) TO WS-OUTPUT-NAME
```

The OCCURS clause specifies how many times the structure repeats. The integer must be a positive literal (not a data-name -- that is what OCCURS DEPENDING ON is for).

### OCCURS DEPENDING ON

OCCURS DEPENDING ON (ODO) defines a variable-length table whose current logical size is determined by a data-name at runtime.

```cobol
       01  ORDER-RECORD.
           05  ORDER-ID            PIC X(10).
           05  LINE-COUNT          PIC 9(03).
           05  ORDER-LINE OCCURS 1 TO 500 TIMES
                          DEPENDING ON LINE-COUNT.
               10  ITEM-CODE       PIC X(08).
               10  ITEM-QTY        PIC 9(05).
               10  ITEM-PRICE      PIC 9(05)V99.
```

The minimum (1) and maximum (500) set the bounds. The compiler allocates space for 500 entries, but when you WRITE or move the record, only LINE-COUNT entries are processed. Changing LINE-COUNT immediately changes the logical table size.

Rules for ODO:
- The ODO object (LINE-COUNT) must be a positive integer data item.
- The ODO object should not be subordinate to the ODO subject, nor should it follow the ODO subject in the same record if that record is used for I/O, as this can cause unpredictable results.
- The minimum value can be 0 (allowing an empty table) or any positive integer up to the maximum.
- Nested ODO (an ODO table containing another ODO table) is permitted by some compilers but is complex and error-prone.

### INDEXED BY Clause

The INDEXED BY clause declares one or more index-names associated with a table. These indexes exist only in the context of the table and are not declared elsewhere in the DATA DIVISION.

```cobol
       01  STATE-TABLE.
           05  STATE-ENTRY OCCURS 50 TIMES
                           ASCENDING KEY IS STATE-CODE
                           INDEXED BY ST-IDX.
               10  STATE-CODE      PIC X(02).
               10  STATE-NAME      PIC X(20).
               10  STATE-TAX-RATE  PIC V9(04).
```

Multiple indexes can be declared on one table:

```cobol
           05  PRODUCT-ENTRY OCCURS 1000 TIMES
                             INDEXED BY PR-IDX PR-SAVE-IDX.
```

This is useful when you need to save an index position (using `SET PR-SAVE-IDX TO PR-IDX`) before doing a secondary search.

### SET Statement

The SET statement is the primary way to manipulate index-names. It has several forms:

**Setting an index to a value:**
```cobol
       SET ST-IDX TO 1
```
This sets the index to point to occurrence 1 (internally storing the byte offset for the first element).

**Setting an index from another index:**
```cobol
       SET PR-SAVE-IDX TO PR-IDX
```

**Incrementing or decrementing an index:**
```cobol
       SET ST-IDX UP BY 1
       SET ST-IDX DOWN BY 3
```
UP BY and DOWN BY adjust by the specified number of occurrences (not bytes).

**Converting an index to a numeric value:**
```cobol
       SET WS-OCCURRENCE-NUM TO ST-IDX
```
Where WS-OCCURRENCE-NUM is an ordinary numeric integer item. This converts the byte displacement back to an occurrence number.

**Setting an index from a numeric value:**
```cobol
       SET ST-IDX TO WS-OCCURRENCE-NUM
```

Important: You cannot use MOVE, ADD, or SUBTRACT with index-names. Always use SET.

### SEARCH (Serial Search)

The SEARCH statement performs a linear (sequential) search through a table. It requires the table to have an INDEXED BY clause. SEARCH automatically increments the associated index.

```cobol
       SET ST-IDX TO 1
       SEARCH STATE-ENTRY
           AT END
               MOVE 'STATE NOT FOUND' TO WS-MESSAGE
           WHEN STATE-CODE(ST-IDX) = WS-INPUT-STATE
               MOVE STATE-NAME(ST-IDX) TO WS-OUTPUT-NAME
               MOVE STATE-TAX-RATE(ST-IDX) TO WS-OUTPUT-RATE
       END-SEARCH
```

Key rules for SEARCH:
- You must initialize the index before the SEARCH (typically `SET idx TO 1`). SEARCH does not reset the index automatically; it begins searching from the current index position.
- The WHEN clause specifies the condition to match. Multiple WHEN clauses are permitted; the first one that is true causes execution of its imperative statement, and the search ends.
- AT END executes if the index goes past the last element without any WHEN condition being satisfied.
- SEARCH increments the index by 1 after each unsuccessful test.
- For variable-length tables (ODO), SEARCH respects the current value of the DEPENDING ON object.

### SEARCH ALL (Binary Search)

SEARCH ALL performs a binary search, which is dramatically faster for large tables. It requires ASCENDING KEY or DESCENDING KEY to be declared on the OCCURS clause and the table data to be pre-sorted in that key order.

```cobol
       SEARCH ALL STATE-ENTRY
           AT END
               MOVE 'STATE NOT FOUND' TO WS-MESSAGE
           WHEN STATE-CODE(ST-IDX) = WS-INPUT-STATE
               MOVE STATE-NAME(ST-IDX) TO WS-OUTPUT-NAME
               MOVE STATE-TAX-RATE(ST-IDX) TO WS-OUTPUT-RATE
       END-SEARCH
```

Key rules for SEARCH ALL:
- You do **not** need to initialize the index before SEARCH ALL. The compiler manages the index internally to implement the binary search algorithm.
- Only **one** WHEN clause is permitted (unlike SEARCH which allows multiple).
- The WHEN condition must test the declared key field(s) using `=` (equality) only. Relational tests other than `=` are not allowed.
- Compound conditions in the WHEN clause must use AND, not OR.
- The left side of each condition in the WHEN clause must reference the key field; the right side must be a literal, data-name, or arithmetic expression that does not reference the table.
- If the table has a composite key (multiple ASCENDING/DESCENDING KEY phrases), all major keys must be tested before minor keys.
- The table data **must** be sorted in the declared key order. If it is not, the results of SEARCH ALL are unpredictable.
- SEARCH ALL has O(log n) time complexity compared to O(n) for SEARCH.

**Example with a composite key:**

```cobol
       01  RATE-TABLE.
           05  RATE-ENTRY OCCURS 500 TIMES
                          ASCENDING KEY IS RT-REGION
                                          RT-CATEGORY
                          INDEXED BY RT-IDX.
               10  RT-REGION       PIC X(03).
               10  RT-CATEGORY     PIC X(02).
               10  RT-AMOUNT       PIC 9(07)V99.

       SEARCH ALL RATE-ENTRY
           AT END
               MOVE ZERO TO WS-RESULT
           WHEN RT-REGION(RT-IDX)   = WS-REGION
            AND RT-CATEGORY(RT-IDX) = WS-CATEGORY
               MOVE RT-AMOUNT(RT-IDX) TO WS-RESULT
       END-SEARCH
```

### REDEFINES with Tables

REDEFINES can be used with tables in two important patterns: pre-loading table data at compile time and providing an alternative view of table storage.

**Pre-loading a table with VALUES via REDEFINES:**

```cobol
       01  MONTH-DATA.
           05  FILLER  PIC X(09) VALUE 'JANUARY  '.
           05  FILLER  PIC X(09) VALUE 'FEBRUARY '.
           05  FILLER  PIC X(09) VALUE 'MARCH    '.
           05  FILLER  PIC X(09) VALUE 'APRIL    '.
           05  FILLER  PIC X(09) VALUE 'MAY      '.
           05  FILLER  PIC X(09) VALUE 'JUNE     '.
           05  FILLER  PIC X(09) VALUE 'JULY     '.
           05  FILLER  PIC X(09) VALUE 'AUGUST   '.
           05  FILLER  PIC X(09) VALUE 'SEPTEMBER'.
           05  FILLER  PIC X(09) VALUE 'OCTOBER  '.
           05  FILLER  PIC X(09) VALUE 'NOVEMBER '.
           05  FILLER  PIC X(09) VALUE 'DECEMBER '.

       01  MONTH-TABLE REDEFINES MONTH-DATA.
           05  MONTH-NAME PIC X(09) OCCURS 12 TIMES.
```

Now `MONTH-NAME(3)` returns `'MARCH    '` without any runtime initialization. This is the classic pattern for static lookup tables in COBOL.

**Alternative view of the same storage:**

```cobol
       01  RAW-RECORD       PIC X(500).
       01  RECORD-AS-TABLE REDEFINES RAW-RECORD.
           05  REC-SEGMENT   PIC X(50) OCCURS 10 TIMES.
```

This lets you treat a flat 500-byte area as 10 segments of 50 bytes each, useful for parsing fixed-format records.

Rules for REDEFINES with OCCURS:
- An item with an OCCURS clause cannot be the subject of a REDEFINES at the same level, but it can be subordinate to a REDEFINES item.
- The REDEFINES and the original item must be at the same level and occupy the same amount of storage.
- REDEFINES is not allowed on an item that has OCCURS DEPENDING ON, nor on items subordinate to an ODO item in certain contexts.

### Multi-Dimensional Table Definition

```cobol
       01  SALES-DATA.
           05  REGION-ENTRY OCCURS 4 TIMES
                            INDEXED BY RG-IDX.
               10  REGION-CODE     PIC X(02).
               10  QUARTER-ENTRY OCCURS 4 TIMES
                                 INDEXED BY QT-IDX.
                   15  QTR-SALES   PIC 9(09)V99.
                   15  QTR-RETURNS PIC 9(07)V99.
```

To reference a specific value:

```cobol
       MOVE QTR-SALES(3, 2) TO WS-AMOUNT
```

This retrieves the sales for region 3, quarter 2. The outermost dimension is listed first.

**Three-dimensional example:**

```cobol
       01  BUDGET-CUBE.
           05  DEPT-ENTRY OCCURS 10 TIMES
                          INDEXED BY DP-IDX.
               10  MONTH-ENTRY OCCURS 12 TIMES
                               INDEXED BY MN-IDX.
                   15  CATEGORY-ENTRY OCCURS 5 TIMES
                                      INDEXED BY CT-IDX.
                       20  BUDGET-AMT  PIC 9(09)V99.
```

Reference: `BUDGET-AMT(DP-IDX, MN-IDX, CT-IDX)` or `BUDGET-AMT(3, 6, 2)`.

To process an entire multi-dimensional table, nest PERFORM loops from outermost to innermost:

```cobol
       PERFORM VARYING DP-IDX FROM 1 BY 1
                        UNTIL DP-IDX > 10
           PERFORM VARYING MN-IDX FROM 1 BY 1
                            UNTIL MN-IDX > 12
               PERFORM VARYING CT-IDX FROM 1 BY 1
                                UNTIL CT-IDX > 5
                   ADD BUDGET-AMT(DP-IDX, MN-IDX, CT-IDX)
                       TO WS-GRAND-TOTAL
               END-PERFORM
           END-PERFORM
       END-PERFORM
```

## Common Patterns

### Pattern 1: Load-and-Lookup Table

The most common table pattern in production COBOL is loading reference data from a file or database into a table at program initialization, then using SEARCH or SEARCH ALL to look up values during processing.

```cobol
       WORKING-STORAGE SECTION.
       01  WS-TABLE-LOADED-FLAG  PIC X VALUE 'N'.
           88  TABLE-IS-LOADED   VALUE 'Y'.
       01  WS-TABLE-COUNT        PIC 9(04) VALUE ZERO.
       01  PRODUCT-TABLE.
           05  PROD-ENTRY OCCURS 1 TO 5000 TIMES
                          DEPENDING ON WS-TABLE-COUNT
                          ASCENDING KEY IS PROD-CODE
                          INDEXED BY PD-IDX.
               10  PROD-CODE      PIC X(10).
               10  PROD-DESC      PIC X(40).
               10  PROD-PRICE     PIC 9(05)V99.

       PROCEDURE DIVISION.
       0000-MAIN.
           PERFORM 1000-LOAD-TABLE
           PERFORM 2000-PROCESS-TRANSACTIONS
               UNTIL END-OF-INPUT
           STOP RUN.

       1000-LOAD-TABLE.
           OPEN INPUT PRODUCT-FILE
           PERFORM UNTIL END-OF-PRODUCT-FILE
               READ PRODUCT-FILE INTO WS-PRODUCT-REC
                   AT END SET END-OF-PRODUCT-FILE TO TRUE
                   NOT AT END
                       ADD 1 TO WS-TABLE-COUNT
                       MOVE PR-CODE TO PROD-CODE(WS-TABLE-COUNT)
                       MOVE PR-DESC TO PROD-DESC(WS-TABLE-COUNT)
                       MOVE PR-PRICE TO PROD-PRICE(WS-TABLE-COUNT)
               END-READ
           END-PERFORM
           CLOSE PRODUCT-FILE
           SET TABLE-IS-LOADED TO TRUE.
```

### Pattern 2: Accumulator Table

Tables used to accumulate totals by category, such as monthly totals or department totals.

```cobol
       01  MONTHLY-TOTALS.
           05  MONTH-ACCUM OCCURS 12 TIMES
                           INDEXED BY MA-IDX.
               10  MONTH-SALES     PIC 9(11)V99 VALUE ZERO.
               10  MONTH-COST      PIC 9(11)V99 VALUE ZERO.
               10  MONTH-PROFIT    PIC S9(11)V99 VALUE ZERO.

      * During processing:
           ADD TRANS-AMOUNT TO MONTH-SALES(TRANS-MONTH)
```

### Pattern 3: Static Lookup with REDEFINES

Used for small, unchanging reference data such as day names, month names, or state abbreviations. See the REDEFINES with Tables section above for the full example.

### Pattern 4: Table as Output Buffer

Building a detail line with repeating columns:

```cobol
       01  REPORT-LINE.
           05  RL-DEPT-NAME       PIC X(15).
           05  RL-QUARTER OCCURS 4 TIMES.
               10  FILLER         PIC X(02) VALUE SPACES.
               10  RL-QTR-AMT    PIC ZZZ,ZZZ,ZZ9.99.
```

### Pattern 5: Parallel Tables

Two or more tables with the same number of entries where elements at the same position are logically related. While this works, a single table with a composite structure (grouping related fields into one OCCURS) is generally preferred for clarity and maintainability.

### Pattern 6: Table Sorting

COBOL does not have a built-in table SORT statement in all dialects. When available (some compilers offer SORT on tables directly), it is convenient. Otherwise, programmers implement bubble sort or other algorithms manually:

```cobol
       PERFORM VARYING WS-OUTER FROM 1 BY 1
                        UNTIL WS-OUTER >= WS-TABLE-COUNT
           PERFORM VARYING WS-INNER FROM WS-OUTER BY 1
                            UNTIL WS-INNER > WS-TABLE-COUNT
               IF SORT-KEY(WS-INNER) < SORT-KEY(WS-OUTER)
                   MOVE TABLE-ROW(WS-INNER) TO WS-TEMP-ROW
                   MOVE TABLE-ROW(WS-OUTER) TO TABLE-ROW(WS-INNER)
                   MOVE WS-TEMP-ROW TO TABLE-ROW(WS-OUTER)
               END-IF
           END-PERFORM
       END-PERFORM
```

## Gotchas

- **Subscript/index out of bounds.** COBOL does not perform bounds checking by default. Accessing element 101 of a 100-element table silently reads or writes adjacent memory, causing data corruption or abends. Some compilers offer a compile option (e.g., SSRANGE on IBM) to enable runtime bounds checking, but this adds overhead and is typically used only during testing.

- **Forgetting to SET the index before SEARCH.** SEARCH (serial) does not reset the index to 1 automatically. If you forget `SET idx TO 1` before the SEARCH, you will start searching from wherever the index was last left, potentially missing matches or immediately hitting AT END.

- **Using SEARCH ALL on unsorted data.** SEARCH ALL assumes the table is sorted in the declared key order. If the data is not actually sorted, the binary search algorithm may skip over matching entries and return false negatives. There is no runtime error -- it simply does not find the record.

- **Confusing subscript and index semantics.** You cannot MOVE a value to an index-name or ADD to an index-name. These compile without error on some legacy compilers but produce incorrect results because indexes store byte offsets, not occurrence numbers. Always use SET.

- **OCCURS DEPENDING ON object placement.** The ODO object (the counter) should not appear after the ODO subject in the same record for I/O operations. Some compilers allow it in WORKING-STORAGE, but for records used in READ/WRITE, placing the counter after the variable-length area causes unpredictable record lengths.

- **REDEFINES storage mismatch.** When using REDEFINES to overlay a table, the total byte count of the REDEFINES subject must exactly match the original item. If MONTH-DATA is 108 bytes (12 x 9), the MONTH-TABLE REDEFINES must also be 108 bytes. A mismatch causes a compile error or, worse, silent data misalignment.

- **VALUE clause on OCCURS items.** In older COBOL standards (COBOL-68, COBOL-74), VALUE clauses are not allowed on items with OCCURS or subordinate to OCCURS. Later standards (COBOL-85 onward) allow VALUE on OCCURS items in WORKING-STORAGE, which initializes every occurrence to the same value. Be aware of which standard your compiler follows.

- **SEARCH ALL with only one WHEN.** Unlike SEARCH which permits multiple WHEN clauses, SEARCH ALL allows exactly one WHEN. Attempting to code multiple WHEN clauses on SEARCH ALL causes a compile error.

- **Multi-dimensional subscript order.** Subscripts are listed outermost to innermost, left to right. Reversing the order is a common bug that compiles fine but accesses completely wrong data. If REGION is the outer OCCURS and QUARTER is the inner, then `AMOUNT(region, quarter)` is correct, not `AMOUNT(quarter, region)`.

- **Performance of SEARCH on large tables.** Serial SEARCH checks elements one at a time. For a 10,000-entry table, this averages 5,000 comparisons per lookup. If the table can be sorted, switching to SEARCH ALL reduces this to about 14 comparisons (log2 of 10,000). For tables accessed in high-volume loops, this difference is significant.

- **INITIALIZE on tables with REDEFINES.** The INITIALIZE statement skips items that are REDEFINES or are subordinate to REDEFINES. If you INITIALIZE a group that contains both the raw data area and its REDEFINES-as-table overlay, only the original item gets initialized. The table view reflects whatever bytes the INITIALIZE set on the original item.

- **ODO with nested tables.** Nesting OCCURS DEPENDING ON inside another OCCURS (or another ODO) is technically permitted by some compilers but creates complex variable-length records where element offsets depend on multiple counters. This is notoriously difficult to get right and should be avoided unless absolutely necessary.

- **Index scope.** Index-names declared via INDEXED BY are associated with their specific table only. You cannot use an index declared on one table to subscript a different table, even if both tables have the same structure. To transfer a position between tables, SET a numeric variable FROM the first index, then SET the second index TO that numeric variable.

## Related Topics

- **data_types.md** -- Understanding PIC clauses and USAGE (DISPLAY, COMP, COMP-3, INDEX) is foundational for defining table element types and for understanding how index storage differs from numeric subscripts.
- **working_storage.md** -- Tables are most commonly defined in WORKING-STORAGE SECTION. That file covers the DATA DIVISION structure, VALUE initialization, and level-number rules that govern where and how OCCURS can appear.
- **conditional_logic.md** -- The WHEN clauses in SEARCH and SEARCH ALL use the same condition syntax (relation conditions, compound conditions, 88-level tests) documented in conditional logic.
- **paragraph_flow.md** -- PERFORM VARYING is the standard mechanism for iterating over tables, especially multi-dimensional ones. Understanding PERFORM nesting and flow control is essential for correct table processing loops.
