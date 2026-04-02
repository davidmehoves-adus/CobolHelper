# Paragraph Flow

## Description
This file covers how execution flows through a COBOL program at the paragraph and section level. It explains every form of the PERFORM statement (inline, out-of-line, TIMES, UNTIL, VARYING, WITH TEST BEFORE/AFTER, and PERFORM THRU), the GO TO statement and its variants, the distinction between sections and paragraphs, fall-through behavior, program termination with STOP RUN and GOBACK, and the EXIT PARAGRAPH / EXIT SECTION statements. Reference this file whenever you need to understand how control passes between paragraphs, how loops work, or how to trace execution order in a COBOL program.

## Table of Contents
- [Core Concepts](#core-concepts)
  - [Paragraphs and Sections](#paragraphs-and-sections)
  - [Fall-Through Behavior](#fall-through-behavior)
  - [Structured vs Unstructured Flow](#structured-vs-unstructured-flow)
  - [Scope of PERFORM](#scope-of-perform)
- [Syntax & Examples](#syntax--examples)
  - [Out-of-Line PERFORM](#out-of-line-perform)
  - [Inline PERFORM](#inline-perform)
  - [PERFORM THRU](#perform-thru)
  - [PERFORM TIMES](#perform-times)
  - [PERFORM UNTIL](#perform-until)
  - [WITH TEST BEFORE and WITH TEST AFTER](#with-test-before-and-with-test-after)
  - [PERFORM VARYING](#perform-varying)
  - [PERFORM VARYING with AFTER](#perform-varying-with-after)
  - [GO TO](#go-to)
  - [GO TO DEPENDING ON](#go-to-depending-on)
  - [ALTER Statement](#alter-statement)
  - [EXIT PARAGRAPH and EXIT SECTION](#exit-paragraph-and-exit-section)
  - [EXIT Statement (Traditional)](#exit-statement-traditional)
  - [STOP RUN](#stop-run)
  - [GOBACK](#goback)
- [Common Patterns](#common-patterns)
- [Gotchas](#gotchas)
- [Related Topics](#related-topics)

## Core Concepts

### Paragraphs and Sections

The PROCEDURE DIVISION is organized into **paragraphs** and optionally **sections**. Understanding the distinction is fundamental to understanding COBOL flow control.

**Paragraph**: A paragraph is a named block of code in the PROCEDURE DIVISION. It begins with a paragraph name followed by a period, and ends implicitly when the next paragraph name, section name, or the end of the PROCEDURE DIVISION is encountered. A paragraph name is a user-defined word coded in Area A (columns 8-11) followed by a period.

```cobol
       PROCEDURE DIVISION.
       0000-MAIN.
           PERFORM 1000-INITIALIZE
           PERFORM 2000-PROCESS
           PERFORM 3000-TERMINATE
           STOP RUN.

       1000-INITIALIZE.
           OPEN INPUT IN-FILE
           OPEN OUTPUT OUT-FILE.

       2000-PROCESS.
           READ IN-FILE
               AT END SET END-OF-FILE TO TRUE
           END-READ
           PERFORM UNTIL END-OF-FILE
               PERFORM 2100-PROCESS-RECORD
               READ IN-FILE
                   AT END SET END-OF-FILE TO TRUE
               END-READ
           END-PERFORM.

       2100-PROCESS-RECORD.
           MOVE IN-REC TO OUT-REC
           WRITE OUT-REC.

       3000-TERMINATE.
           CLOSE IN-FILE
           CLOSE OUT-FILE.
```

**Section**: A section is a named group of one or more paragraphs. It begins with a section name followed by the word SECTION and a period. A section ends when the next section name or the end of the PROCEDURE DIVISION is encountered. All paragraphs between two section headers belong to the first section.

```cobol
       PROCEDURE DIVISION.
       MAIN-LOGIC SECTION.
       0000-MAIN.
           PERFORM INITIALIZATION-LOGIC
           PERFORM PROCESSING-LOGIC
           PERFORM TERMINATION-LOGIC
           STOP RUN.

       INITIALIZATION-LOGIC SECTION.
       1000-INIT-START.
           OPEN INPUT IN-FILE
           OPEN OUTPUT OUT-FILE.

       PROCESSING-LOGIC SECTION.
       2000-PROCESS-START.
           READ IN-FILE
               AT END SET END-OF-FILE TO TRUE
           END-READ.
       2100-PROCESS-LOOP.
           PERFORM UNTIL END-OF-FILE
               PERFORM 2200-PROCESS-RECORD
               READ IN-FILE
                   AT END SET END-OF-FILE TO TRUE
               END-READ
           END-PERFORM.
       2200-PROCESS-RECORD.
           MOVE IN-REC TO OUT-REC
           WRITE OUT-REC.

       TERMINATION-LOGIC SECTION.
       3000-TERM-START.
           CLOSE IN-FILE
           CLOSE OUT-FILE.
```

Key differences between sections and paragraphs:

- When you PERFORM a **section** name, control flows through all paragraphs in that section from beginning to end, then returns to the caller. Every paragraph within the section executes in sequence.
- When you PERFORM a **paragraph** name, only that single paragraph executes (up to the next paragraph or section header), then control returns.
- Sections are required in some contexts (for example, the DECLARATIVES region requires sections, and some compilers require sections when using SORT/MERGE input/output procedures).
- Many modern COBOL shops avoid sections entirely and use only paragraphs, as this gives finer-grained control and reduces the risk of unintended fall-through.

### Fall-Through Behavior

COBOL executes statements sequentially. When one paragraph ends, execution falls through to the next paragraph automatically -- unless a statement like PERFORM, GO TO, STOP RUN, or GOBACK redirects control. This is analogous to fall-through in assembly language: paragraph boundaries are labels, not barriers.

```cobol
       1000-FIRST-PARA.
           DISPLAY "FIRST".
       2000-SECOND-PARA.
           DISPLAY "SECOND".
       3000-THIRD-PARA.
           DISPLAY "THIRD".
           STOP RUN.
```

If execution begins at 1000-FIRST-PARA and simply proceeds, it will display "FIRST", then fall through to 2000-SECOND-PARA and display "SECOND", then fall through to 3000-THIRD-PARA and display "THIRD", and finally stop. The paragraph names do not create walls -- they are entry points.

Fall-through is the single most important concept for understanding legacy COBOL control flow. In structured programs, fall-through is typically avoided by using PERFORM to call paragraphs and returning control to the caller. In unstructured (legacy) programs, fall-through is used intentionally and is a major source of complexity when reading code.

### Structured vs Unstructured Flow

**Structured flow** uses PERFORM as the primary mechanism for transferring control. Each paragraph acts like a subroutine: PERFORM transfers control to it, the paragraph executes, and control returns to the statement after the PERFORM. Programs using structured flow are easier to understand and maintain because every paragraph has a single entry point and a single logical exit point.

**Unstructured flow** relies on GO TO statements and fall-through to move between paragraphs. Control does not automatically return to a calling point -- it simply continues forward or jumps to the GO TO target. This was the dominant style in COBOL programs written before the structured programming movement of the 1970s-1980s. Many legacy programs still use this style.

Most production COBOL systems written since the mid-1980s use structured flow exclusively or predominantly. The GO TO statement is generally discouraged in modern COBOL standards, though it still appears in legacy code and in certain idiomatic patterns (such as GO TO within a PERFORM THRU range to jump to the exit paragraph).

### Scope of PERFORM

When an out-of-line PERFORM transfers control to a paragraph (or range of paragraphs), the runtime establishes a **return point**. When execution reaches the end of the performed range, control returns to the statement following the PERFORM. This is sometimes called the "PERFORM stack" or "PERFORM nesting" mechanism.

COBOL supports nested PERFORMs: a paragraph being PERFORMed may itself PERFORM another paragraph. The runtime tracks each active PERFORM and its return point. However, overlapping PERFORM ranges (where one PERFORM range partially overlaps another active PERFORM range) produce undefined behavior in the COBOL standard and must be avoided.

## Syntax & Examples

### Out-of-Line PERFORM

The most basic PERFORM transfers control to a named paragraph (or section), executes it, and returns.

```cobol
       PERFORM paragraph-name
```

Example:

```cobol
       0000-MAIN.
           PERFORM 1000-OPEN-FILES
           PERFORM 2000-PROCESS-DATA
           PERFORM 3000-CLOSE-FILES
           STOP RUN.
```

When PERFORM 1000-OPEN-FILES executes, control passes to the first statement in 1000-OPEN-FILES. When the end of that paragraph is reached (the next paragraph header or a period ending the last sentence), control returns to the next statement after the PERFORM -- in this case, PERFORM 2000-PROCESS-DATA.

### Inline PERFORM

An inline PERFORM executes one or more statements directly within the PERFORM block without transferring control to another paragraph. It is terminated by END-PERFORM.

```cobol
       PERFORM
           DISPLAY "PROCESSING"
           ADD 1 TO WS-COUNTER
       END-PERFORM
```

A bare inline PERFORM with no TIMES or UNTIL clause executes its body exactly once. This is rarely useful on its own, but inline PERFORM is commonly combined with UNTIL or TIMES to create loops.

```cobol
       PERFORM UNTIL WS-EOF-FLAG = "Y"
           READ INPUT-FILE INTO WS-RECORD
               AT END
                   MOVE "Y" TO WS-EOF-FLAG
               NOT AT END
                   PERFORM 2100-PROCESS-RECORD
           END-READ
       END-PERFORM
```

Inline PERFORMs can be nested:

```cobol
       PERFORM VARYING WS-ROW FROM 1 BY 1
               UNTIL WS-ROW > 10
           PERFORM VARYING WS-COL FROM 1 BY 1
                   UNTIL WS-COL > 10
               COMPUTE WS-TABLE(WS-ROW, WS-COL) =
                   WS-ROW * 10 + WS-COL
           END-PERFORM
       END-PERFORM
```

### PERFORM THRU

PERFORM THRU (or THROUGH -- both spellings are equivalent) executes a range of consecutive paragraphs, starting at the first named paragraph and ending at the last named paragraph.

```cobol
       PERFORM paragraph-1 THRU paragraph-2
```

Control transfers to paragraph-1 and continues sequentially through all paragraphs up to and including paragraph-2. When the end of paragraph-2 is reached, control returns to the statement after the PERFORM.

```cobol
       0000-MAIN.
           PERFORM 1000-INIT THRU 1000-INIT-EXIT
           PERFORM 2000-PROCESS THRU 2000-PROCESS-EXIT
           STOP RUN.

       1000-INIT.
           OPEN INPUT IN-FILE
           OPEN OUTPUT OUT-FILE.
       1000-INIT-EXIT.
           EXIT.

       2000-PROCESS.
           READ IN-FILE
               AT END
                   GO TO 2000-PROCESS-EXIT
           END-READ
           MOVE IN-REC TO OUT-REC
           WRITE OUT-REC.
       2000-PROCESS-EXIT.
           EXIT.
```

The PERFORM THRU pattern is commonly used with an exit paragraph that contains only the EXIT statement. This provides a clean exit point -- a GO TO within the range can jump to the exit paragraph to leave the performed range early. The exit paragraph acts as a collection point; when control reaches it (whether by fall-through or GO TO), the PERFORM return mechanism activates.

### PERFORM TIMES

PERFORM TIMES executes a paragraph (or inline block) a fixed number of times.

```cobol
       PERFORM paragraph-name  integer  TIMES

       PERFORM paragraph-name  identifier  TIMES
```

Examples:

```cobol
      * Out-of-line: perform the paragraph 5 times
           PERFORM 2000-PRINT-LINE 5 TIMES

      * Out-of-line: perform it WS-COUNT times
           PERFORM 2000-PRINT-LINE WS-COUNT TIMES

      * Inline: repeat the block 10 times
           PERFORM 10 TIMES
               DISPLAY "HELLO"
               ADD 1 TO WS-COUNTER
           END-PERFORM
```

If the count is zero or negative, the body does not execute at all. The count is evaluated once at the start of the PERFORM; changes to the identifier during execution do not affect the iteration count.

### PERFORM UNTIL

PERFORM UNTIL repeats execution of a paragraph or inline block until a condition becomes true.

```cobol
       PERFORM paragraph-name UNTIL condition

       PERFORM UNTIL condition
           statements
       END-PERFORM
```

By default, PERFORM UNTIL tests the condition **before** each iteration (WITH TEST BEFORE is the default). If the condition is already true when the PERFORM is first reached, the body never executes.

```cobol
      * Out-of-line: read and process until end of file
           PERFORM 2000-READ-AND-PROCESS
               UNTIL WS-EOF = "Y"

      * Inline: accumulate until total exceeds limit
           PERFORM UNTIL WS-TOTAL > 1000
               ADD WS-AMOUNT TO WS-TOTAL
               ADD 1 TO WS-COUNT
           END-PERFORM
```

The condition is re-evaluated before each iteration. It may be any valid COBOL conditional expression, including compound conditions:

```cobol
           PERFORM 2000-PROCESS
               UNTIL WS-EOF = "Y"
               OR WS-ERROR-COUNT > 10
```

### WITH TEST BEFORE and WITH TEST AFTER

The WITH TEST phrase controls whether the loop condition is tested before or after each iteration.

```cobol
       PERFORM paragraph-name
           WITH TEST BEFORE
           UNTIL condition

       PERFORM paragraph-name
           WITH TEST AFTER
           UNTIL condition
```

**WITH TEST BEFORE** (the default): The condition is evaluated before the first and each subsequent execution. If the condition is true initially, the body never executes. This is analogous to a `while` loop in other languages.

**WITH TEST AFTER**: The condition is evaluated after each execution. The body always executes at least once, regardless of the initial condition. This is analogous to a `do...while` loop.

```cobol
      * TEST BEFORE (default): may execute zero times
           PERFORM 2000-PROCESS
               WITH TEST BEFORE
               UNTIL WS-COUNT > 10

      * TEST AFTER: always executes at least once
           PERFORM 2000-PROCESS
               WITH TEST AFTER
               UNTIL WS-COUNT > 10

      * Inline with TEST AFTER
           PERFORM WITH TEST AFTER
               UNTIL WS-RESPONSE = "Q"
               DISPLAY "ENTER CHOICE (Q TO QUIT): "
               ACCEPT WS-RESPONSE
           END-PERFORM
```

The TEST AFTER form is particularly useful for menu-driven loops and read-process loops where you need to execute the body before you can test the condition for the first time.

### PERFORM VARYING

PERFORM VARYING automatically increments (or decrements) an identifier through a range of values. It combines initialization, testing, and incrementing into a single statement.

```cobol
       PERFORM paragraph-name
           VARYING identifier-1 FROM value-1 BY value-2
           UNTIL condition
```

The logic is:

1. Set identifier-1 to value-1 (the FROM value).
2. Evaluate the UNTIL condition (assuming TEST BEFORE, the default).
3. If the condition is true, exit the loop.
4. Execute the paragraph (or inline body).
5. Add value-2 (the BY value) to identifier-1.
6. Go to step 2.

```cobol
      * Count from 1 to 100
           PERFORM 2000-PROCESS
               VARYING WS-INDEX FROM 1 BY 1
               UNTIL WS-INDEX > 100

      * Count down from 10 to 1
           PERFORM 2000-PROCESS
               VARYING WS-INDEX FROM 10 BY -1
               UNTIL WS-INDEX < 1

      * Inline VARYING
           PERFORM VARYING WS-I FROM 1 BY 1
                   UNTIL WS-I > WS-TABLE-SIZE
               DISPLAY WS-TABLE-ENTRY(WS-I)
           END-PERFORM
```

The FROM, BY, and UNTIL values can be identifiers or literals. If they are identifiers, their values are evaluated:
- FROM: evaluated once at the start.
- BY: evaluated each time the increment occurs.
- UNTIL: the condition is evaluated each iteration.

PERFORM VARYING also supports WITH TEST BEFORE/AFTER:

```cobol
           PERFORM VARYING WS-I FROM 1 BY 1
               WITH TEST AFTER
               UNTIL WS-I >= WS-MAX
               DISPLAY WS-TABLE-ENTRY(WS-I)
           END-PERFORM
```

### PERFORM VARYING with AFTER

For nested iteration (analogous to nested for-loops), PERFORM VARYING supports the AFTER phrase. This allows a single PERFORM statement to iterate over multiple dimensions.

```cobol
       PERFORM paragraph-name
           VARYING identifier-1 FROM value-1 BY value-2
               UNTIL condition-1
           AFTER identifier-2 FROM value-3 BY value-4
               UNTIL condition-2
```

The outer identifier (identifier-1) changes slowly, and the inner identifier (identifier-2 in the AFTER clause) changes rapidly. For each value of identifier-1, identifier-2 cycles through its full range.

```cobol
      * Process a 10x5 table
           PERFORM 3000-PROCESS-CELL
               VARYING WS-ROW FROM 1 BY 1
                   UNTIL WS-ROW > 10
               AFTER WS-COL FROM 1 BY 1
                   UNTIL WS-COL > 5
```

This executes 3000-PROCESS-CELL 50 times: WS-ROW goes from 1 to 10, and for each value of WS-ROW, WS-COL goes from 1 to 5.

Multiple AFTER phrases can be chained for three or more dimensions, though this is uncommon:

```cobol
           PERFORM 4000-PROCESS-ELEMENT
               VARYING WS-X FROM 1 BY 1 UNTIL WS-X > 3
               AFTER   WS-Y FROM 1 BY 1 UNTIL WS-Y > 4
               AFTER   WS-Z FROM 1 BY 1 UNTIL WS-Z > 5
```

Note: The AFTER phrase is only available with out-of-line PERFORM (where a paragraph name is specified). You cannot use AFTER with inline PERFORM. For inline nested iteration, use nested inline PERFORM statements instead.

### GO TO

The GO TO statement transfers control unconditionally to a specified paragraph or section. Unlike PERFORM, it does not establish a return point.

```cobol
       GO TO paragraph-name
```

Example:

```cobol
       2000-PROCESS.
           READ IN-FILE
               AT END
                   GO TO 2000-PROCESS-EXIT
           END-READ
           MOVE IN-REC TO OUT-REC
           WRITE OUT-REC
           GO TO 2000-PROCESS.
       2000-PROCESS-EXIT.
           EXIT.
```

In this example, GO TO 2000-PROCESS creates a loop by jumping back to the beginning of the paragraph. GO TO 2000-PROCESS-EXIT provides an early exit from the loop when the end of file is reached. This is the classic unstructured COBOL looping pattern that predates the PERFORM UNTIL construct.

When GO TO is used within a PERFORM THRU range, it must jump to a location within that range. Jumping outside the PERFORM range results in undefined behavior because the PERFORM return mechanism is bypassed.

### GO TO DEPENDING ON

GO TO DEPENDING ON provides a computed branch, selecting one of several target paragraphs based on the value of a numeric identifier.

```cobol
       GO TO paragraph-1
              paragraph-2
              paragraph-3
              DEPENDING ON identifier
```

If identifier is 1, control goes to paragraph-1. If 2, to paragraph-2. If 3, to paragraph-3. If the value is less than 1 or greater than the number of paragraphs listed, the GO TO has no effect and execution falls through to the next statement.

```cobol
       2000-ROUTE-TRANSACTION.
           GO TO 2100-ADD-RECORD
                  2200-UPDATE-RECORD
                  2300-DELETE-RECORD
                  2400-INQUIRY
               DEPENDING ON WS-TRANS-CODE
           DISPLAY "INVALID TRANSACTION CODE"
           GO TO 2000-ROUTE-EXIT.

       2100-ADD-RECORD.
           ...
```

In modern COBOL, EVALUATE is strongly preferred over GO TO DEPENDING ON because it is more readable, does not rely on numeric positions, and does not require unstructured jumps.

### ALTER Statement

The ALTER statement changes the target of a GO TO statement at runtime. It exists in the COBOL standard but is universally considered obsolete and dangerous. It is mentioned here only for completeness when reading very old programs.

```cobol
       ALTER paragraph-name-1 TO PROCEED TO paragraph-name-2
```

This changes a GO TO in paragraph-name-1 so that it branches to paragraph-name-2 instead of its original target. The paragraph being altered must contain only a single GO TO statement.

```cobol
       SETUP-PARA.
           ALTER ROUTING-PARA TO PROCEED TO PROCESS-B.

       ROUTING-PARA.
           GO TO PROCESS-A.

       PROCESS-A.
           DISPLAY "PATH A".
       PROCESS-B.
           DISPLAY "PATH B".
```

After SETUP-PARA executes, ROUTING-PARA's GO TO will branch to PROCESS-B instead of PROCESS-A. This is extremely difficult to follow in large programs and should never be used in new code. Many modern compilers flag it with warnings, and it has been marked as obsolete in the COBOL standard since COBOL-85.

### EXIT PARAGRAPH and EXIT SECTION

EXIT PARAGRAPH and EXIT SECTION provide structured ways to leave the current paragraph or section early, without using GO TO.

```cobol
       EXIT PARAGRAPH
       EXIT SECTION
```

**EXIT PARAGRAPH** transfers control to the implicit end of the current paragraph. If the paragraph is being PERFORMed, control returns to the caller. If it is executing via fall-through, control passes to the next paragraph.

```cobol
       2000-VALIDATE-RECORD.
           IF WS-RECORD-TYPE = SPACES
               EXIT PARAGRAPH
           END-IF
           IF WS-AMOUNT NOT NUMERIC
               MOVE "INVALID AMOUNT" TO WS-ERROR-MSG
               PERFORM 9000-LOG-ERROR
               EXIT PARAGRAPH
           END-IF
           PERFORM 2100-PROCESS-VALID-RECORD.
```

**EXIT SECTION** transfers control to the implicit end of the current section. All remaining paragraphs in the section are skipped.

```cobol
       VALIDATION SECTION.
       3000-CHECK-HEADER.
           IF WS-HEADER-INVALID
               MOVE "BAD HEADER" TO WS-ERROR-MSG
               EXIT SECTION
           END-IF.
       3100-CHECK-DETAIL.
           IF WS-DETAIL-INVALID
               MOVE "BAD DETAIL" TO WS-ERROR-MSG
               EXIT SECTION
           END-IF.
       3200-CHECK-TRAILER.
           CONTINUE.
```

EXIT PARAGRAPH and EXIT SECTION are available in COBOL-2002 and later, and are supported as extensions by most major compilers (IBM Enterprise COBOL, Micro Focus, etc.) even in earlier standard modes. They are the modern, structured alternative to using GO TO to jump to an exit paragraph.

### EXIT Statement (Traditional)

The traditional EXIT statement (without PARAGRAPH or SECTION qualifier) is a no-operation statement. It generates no executable code. It exists solely as a placeholder to give a paragraph a name and a body.

```cobol
       2000-PROCESS-EXIT.
           EXIT.
```

This pattern is used as the endpoint of a PERFORM THRU range. The exit paragraph provides a target for GO TO statements within the range and a clear end-of-range marker. The EXIT statement is required because a paragraph must contain at least one statement.

Note: EXIT by itself does not cause control to return to a PERFORM caller. Control returns when execution reaches the end of the PERFORMed range, which happens naturally when the exit paragraph (at the end of the THRU range) finishes executing.

### STOP RUN

STOP RUN terminates the entire program (the run unit). It returns control to the operating system or the invoking environment.

```cobol
       STOP RUN
```

Key behaviors:
- All open files are closed (implicitly by the runtime).
- All resources held by the program are released.
- In a main program, STOP RUN is the standard termination statement.
- In a subprogram called via CALL, STOP RUN terminates the entire run unit, including the calling program. This is almost always undesirable in a subprogram.
- No return code is passed back to the caller unless a vendor-specific mechanism is used (such as MOVE value TO RETURN-CODE before STOP RUN on IBM mainframes).

```cobol
       0000-MAIN.
           PERFORM 1000-INITIALIZE
           PERFORM 2000-PROCESS
           PERFORM 3000-TERMINATE
           STOP RUN.
```

### GOBACK

GOBACK returns control to the calling program or the operating system, depending on context.

```cobol
       GOBACK
```

Key behaviors:
- In a **main program**: GOBACK is functionally equivalent to STOP RUN. It terminates the run unit and returns to the OS.
- In a **subprogram** (called via CALL): GOBACK returns control to the calling program at the statement following the CALL. The subprogram's state may or may not be preserved depending on whether it is defined as INITIAL.
- GOBACK is the preferred termination statement for subprograms because it does not terminate the entire run unit.
- Many shops standardize on GOBACK for all programs (both main and sub) because it behaves correctly in either context.

```cobol
      * In a subprogram
       PROCEDURE DIVISION USING LS-INPUT-RECORD
                                LS-OUTPUT-RECORD.
       0000-MAIN.
           PERFORM 1000-VALIDATE
           PERFORM 2000-TRANSFORM
           GOBACK.
```

STOP RUN vs GOBACK summary:

| Statement | In Main Program   | In Subprogram         |
|-----------|-------------------|-----------------------|
| STOP RUN  | Terminates run unit | Terminates entire run unit (dangerous) |
| GOBACK    | Terminates run unit | Returns to caller (safe) |

## Common Patterns

**Main-driver pattern with paragraphs**: The most common structured COBOL pattern uses a main paragraph that PERFORMs other paragraphs in sequence. Each paragraph has a single responsibility. No fall-through occurs because STOP RUN ends the main paragraph before execution could fall into subordinate paragraphs.

```cobol
       0000-MAIN.
           PERFORM 1000-INITIALIZE
           PERFORM 2000-PROCESS-FILE
           PERFORM 3000-TERMINATE
           STOP RUN.

       1000-INITIALIZE.
           OPEN INPUT IN-FILE
           OPEN OUTPUT OUT-FILE
           PERFORM 1100-READ-INPUT.

       1100-READ-INPUT.
           READ IN-FILE INTO WS-INPUT-REC
               AT END SET WS-EOF TO TRUE
           END-READ.

       2000-PROCESS-FILE.
           PERFORM UNTIL WS-EOF
               PERFORM 2100-PROCESS-RECORD
               PERFORM 1100-READ-INPUT
           END-PERFORM.

       2100-PROCESS-RECORD.
           MOVE WS-INPUT-REC TO WS-OUTPUT-REC
           WRITE OUT-REC FROM WS-OUTPUT-REC.

       3000-TERMINATE.
           CLOSE IN-FILE
           CLOSE OUT-FILE.
```

**PERFORM THRU with exit paragraph**: Legacy and some modern programs use PERFORM THRU with a paired exit paragraph. GO TO statements within the range jump to the exit paragraph for early return.

```cobol
       0000-MAIN.
           PERFORM 2000-VALIDATE THRU 2000-VALIDATE-EXIT.

       2000-VALIDATE.
           IF WS-FIELD-A = SPACES
               MOVE "FIELD A MISSING" TO WS-ERROR
               GO TO 2000-VALIDATE-EXIT
           END-IF
           IF WS-FIELD-B NOT NUMERIC
               MOVE "FIELD B NOT NUMERIC" TO WS-ERROR
               GO TO 2000-VALIDATE-EXIT
           END-IF
           MOVE "VALID" TO WS-STATUS.
       2000-VALIDATE-EXIT.
           EXIT.
```

**Read-process loop with priming read**: The standard file-processing loop reads the first record before entering the loop, then reads subsequent records at the end of each iteration.

```cobol
           READ IN-FILE INTO WS-RECORD
               AT END SET WS-EOF TO TRUE
           END-READ
           PERFORM UNTIL WS-EOF
               PERFORM 2100-PROCESS-RECORD
               READ IN-FILE INTO WS-RECORD
                   AT END SET WS-EOF TO TRUE
               END-READ
           END-PERFORM
```

**Inline PERFORM for simple loops**: When the loop body is short and calling a separate paragraph would be excessive, inline PERFORM keeps the logic local and readable.

```cobol
           MOVE 0 TO WS-TOTAL
           PERFORM VARYING WS-I FROM 1 BY 1
                   UNTIL WS-I > WS-ITEM-COUNT
               ADD WS-AMOUNT(WS-I) TO WS-TOTAL
           END-PERFORM
```

**Nested PERFORM for table processing**: Multi-dimensional tables are processed with nested PERFORMs or PERFORM VARYING with AFTER.

```cobol
      * Using nested inline PERFORMs (modern style)
           PERFORM VARYING WS-ROW FROM 1 BY 1
                   UNTIL WS-ROW > WS-MAX-ROWS
               PERFORM VARYING WS-COL FROM 1 BY 1
                       UNTIL WS-COL > WS-MAX-COLS
                   DISPLAY WS-CELL(WS-ROW, WS-COL)
               END-PERFORM
           END-PERFORM

      * Using PERFORM VARYING with AFTER (traditional style)
           PERFORM 5000-DISPLAY-CELL
               VARYING WS-ROW FROM 1 BY 1
                   UNTIL WS-ROW > WS-MAX-ROWS
               AFTER WS-COL FROM 1 BY 1
                   UNTIL WS-COL > WS-MAX-COLS
```

**Early exit with EXIT PARAGRAPH**: Modern programs use EXIT PARAGRAPH instead of GO TO for early return from a paragraph. This avoids the need for PERFORM THRU and an exit paragraph.

```cobol
       2000-VALIDATE.
           IF WS-FIELD-A = SPACES
               MOVE "FIELD A MISSING" TO WS-ERROR
               EXIT PARAGRAPH
           END-IF
           IF WS-FIELD-B NOT NUMERIC
               MOVE "FIELD B NOT NUMERIC" TO WS-ERROR
               EXIT PARAGRAPH
           END-IF
           MOVE "VALID" TO WS-STATUS.
```

**Section-based flow for SORT procedures**: Some compilers require input and output procedures for the SORT statement to be sections.

```cobol
           SORT SORT-FILE
               ON ASCENDING KEY SORT-KEY
               INPUT PROCEDURE IS INPUT-SECTION
               OUTPUT PROCEDURE IS OUTPUT-SECTION.

       INPUT-SECTION SECTION.
       4000-INPUT-START.
           PERFORM UNTIL WS-EOF
               RELEASE SORT-REC FROM WS-INPUT-REC
               READ IN-FILE INTO WS-INPUT-REC
                   AT END SET WS-EOF TO TRUE
               END-READ
           END-PERFORM.

       OUTPUT-SECTION SECTION.
       5000-OUTPUT-START.
           PERFORM UNTIL WS-SORT-EOF
               RETURN SORT-FILE INTO WS-OUTPUT-REC
                   AT END SET WS-SORT-EOF TO TRUE
                   NOT AT END
                       WRITE OUT-REC FROM WS-OUTPUT-REC
               END-RETURN
           END-PERFORM.
```

## Gotchas

- **Fall-through after a PERFORMed paragraph can cause silent bugs.** If you PERFORM 1000-PARA and 1000-PARA does not end with a period or explicit scope terminator, but the code at the top of 2000-PARA accidentally becomes part of 1000-PARA (for example, due to a missing period), the PERFORMed range extends silently into the next paragraph. The program compiles and runs but produces wrong results. Always verify paragraph boundaries carefully.

- **Overlapping PERFORM ranges produce undefined behavior.** If paragraph A PERFORMs B THRU D, and paragraph C (within that range) PERFORMs B THRU D again, the PERFORM ranges overlap. The COBOL standard says the behavior is undefined. In practice, this can cause infinite loops, skipped code, or crashes. Never allow one active PERFORM range to overlap another.

- **GO TO out of a PERFORM range corrupts the PERFORM stack.** If you PERFORM 1000-PARA THRU 1000-EXIT, and code within that range executes GO TO 9000-ABORT (outside the range), the PERFORM return mechanism is never triggered. The runtime's PERFORM stack becomes inconsistent, which can cause unpredictable behavior later. Always keep GO TO targets within the PERFORM THRU range.

- **STOP RUN in a subprogram terminates everything.** If a subprogram called via CALL executes STOP RUN, the entire run unit ends -- including the calling program and all programs up the call chain. Use GOBACK in subprograms instead of STOP RUN.

- **PERFORM TIMES evaluates the count once.** The iteration count for PERFORM n TIMES is captured at the beginning of the loop. Modifying the counter variable inside the loop body does not change the number of iterations. This sometimes surprises programmers who try to exit early by changing the counter.

- **PERFORM VARYING final value is one step beyond the limit.** After a PERFORM VARYING WS-I FROM 1 BY 1 UNTIL WS-I > 10 completes, WS-I contains 11, not 10. The incrementing occurs before the test in the next iteration, so the identifier ends up at the first value that satisfies the UNTIL condition. Code that uses the identifier after the loop must account for this.

- **WITH TEST AFTER always executes at least once.** If you use WITH TEST AFTER and the condition is already true before the first iteration, the body still executes once. This can cause errors if the body assumes the condition is false (for example, processing a record when the file is already at end-of-file).

- **Missing period can extend a paragraph unexpectedly.** In older COBOL style (before END-IF, END-PERFORM, etc.), the period terminates statements. A missing period can cause an IF statement to encompass the next paragraph's code. Even in modern COBOL with explicit scope terminators, the paragraph-ending period is still required and its absence can cause the paragraph boundary to shift.

- **PERFORM with no UNTIL or TIMES on an out-of-line paragraph executes exactly once.** This is correct behavior but occasionally confuses programmers who expect the paragraph to loop. If you want a loop, you must include UNTIL or TIMES.

- **EXIT by itself does nothing.** The EXIT statement (without PARAGRAPH or SECTION) is a no-op. It does not cause a return from the current PERFORM. It is only useful as a placeholder to give a paragraph a body. Control returns to the PERFORM caller when execution reaches the end of the PERFORMed range, which happens because the EXIT paragraph is the last paragraph in the THRU range, not because EXIT has any special return behavior.

- **Sections cause entire section to execute on PERFORM.** When you PERFORM a section name, all paragraphs within that section execute -- not just the first paragraph. If you intended to execute only the first paragraph, you must PERFORM the paragraph name, not the section name. Confusing a section name with a paragraph name can cause significantly more code to execute than expected.

- **AFTER clause is not available for inline PERFORM.** Attempting to use the AFTER clause with inline PERFORM (END-PERFORM) will cause a compilation error. Use nested inline PERFORMs or an out-of-line PERFORM with AFTER for multi-dimensional iteration.

- **ALTER makes programs nearly impossible to debug.** The ALTER statement changes GO TO targets at runtime, meaning the control flow shown in the source code does not match the actual execution path. Avoid ALTER in all new code and consider refactoring it out of legacy programs when feasible.

## Related Topics

- **cobol_structure.md** -- Covers the four divisions and overall program structure. The PROCEDURE DIVISION organization into sections and paragraphs is defined at the structural level. Understanding program structure is prerequisite to understanding paragraph flow.
- **conditional_logic.md** -- Conditions used in PERFORM UNTIL and IF statements that control flow within and between paragraphs. The EVALUATE statement is the modern alternative to GO TO DEPENDING ON.
- **subprograms.md** -- CALL transfers control between separately compiled programs. GOBACK vs STOP RUN behavior differs between main programs and subprograms. The PERFORM mechanism is internal to a single program; CALL is the inter-program equivalent.
- **debugging.md** -- Tracing execution flow through paragraphs is a core debugging activity. Understanding PERFORM nesting, GO TO jumps, and fall-through is essential for diagnosing logic errors in COBOL programs.
