# Subprograms

## Description
This file covers COBOL subprogram mechanics: how programs call other programs, pass data between them, and manage subprogram lifecycles. Reference this file when working with the CALL statement, LINKAGE SECTION, inter-program communication, nested programs, or any scenario where one COBOL program invokes another.

## Table of Contents
- [Core Concepts](#core-concepts)
  - [What Is a Subprogram](#what-is-a-subprogram)
  - [Calling Program vs Called Program](#calling-program-vs-called-program)
  - [Static vs Dynamic Calls](#static-vs-dynamic-calls)
  - [Nested Programs](#nested-programs)
  - [COMMON and INITIAL Attributes](#common-and-initial-attributes)
- [Syntax & Examples](#syntax--examples)
  - [The CALL Statement](#the-call-statement)
  - [BY REFERENCE, BY CONTENT, BY VALUE](#by-reference-by-content-by-value)
  - [LINKAGE SECTION](#linkage-section)
  - [PROCEDURE DIVISION USING](#procedure-division-using)
  - [CANCEL Statement](#cancel-statement)
  - [RETURN-CODE Special Register](#return-code-special-register)
  - [ENTRY Statement](#entry-statement)
  - [Calling Conventions](#calling-conventions)
- [Common Patterns](#common-patterns)
- [Gotchas](#gotchas)
- [Related Topics](#related-topics)

---

## Core Concepts

### What Is a Subprogram

A COBOL subprogram is a separately compiled (or nested) program that is invoked by another COBOL program via the `CALL` statement. Subprograms enable modular design, code reuse, and separation of concerns. The calling program transfers control to the subprogram, optionally passes data, and regains control when the subprogram executes `GOBACK` or `STOP RUN`.

Every COBOL subprogram is itself a valid COBOL program with its own four divisions. The key difference is how it is invoked (via `CALL` rather than as a main entry point) and how it receives data (via the `LINKAGE SECTION` and `USING` clause).

### Calling Program vs Called Program

| Aspect | Calling Program | Called Program (Subprogram) |
|---|---|---|
| Initiates the call | Yes, via `CALL` | No |
| Defines data to pass | In WORKING-STORAGE or LOCAL-STORAGE | In LINKAGE SECTION |
| Specifies parameters | `CALL ... USING` | `PROCEDURE DIVISION USING` |
| Returns control | Receives control back after CALL | Via `GOBACK` or `EXIT PROGRAM` |
| Can be called itself | Yes (can be both caller and callee) | Yes |

A single program can act as both a calling program and a called program simultaneously. Deep call chains (A calls B, B calls C, C calls D) are common in production systems.

### Static vs Dynamic Calls

COBOL supports two fundamentally different mechanisms for invoking subprograms:

**Static calls** link the subprogram into the same load module as the calling program at link-edit (bind) time. The subprogram code is physically part of the same executable.

```cobol
       CALL 'SUBPGM1' USING WS-PARAM1 WS-PARAM2
```

When the program name is a literal and the compiler/linker options specify static linking, the call is resolved at compile/link time. The subprogram is always in memory when the calling program is loaded.

**Dynamic calls** resolve the subprogram at runtime. The system searches for the subprogram module when the `CALL` is executed, loads it into memory, and transfers control.

```cobol
       MOVE 'SUBPGM1' TO WS-PGM-NAME
       CALL WS-PGM-NAME USING WS-PARAM1 WS-PARAM2
```

When the program name is specified via a data-name (identifier), the call is always dynamic. When the program name is a literal, whether the call is static or dynamic depends on compiler options (e.g., `DYNAM`/`NODYNAM` on IBM mainframes).

| Aspect | Static Call | Dynamic Call |
|---|---|---|
| Resolution time | Compile/link time | Runtime |
| Program name | Literal (typically) | Identifier or literal with DYNAM |
| Memory | Loaded with caller | Loaded on first CALL |
| CANCEL effect | No effect (cannot be unloaded) | Releases memory, resets state |
| Performance | Slightly faster (no load overhead) | Slightly slower (load + search) |
| Flexibility | Must relink to change subprogram | Can swap subprograms at runtime |
| Memory usage | Always in memory | Can be released via CANCEL |

### Nested Programs

COBOL allows programs to be physically nested inside other programs. A nested program is coded between the `IDENTIFICATION DIVISION` of the containing program and its `END PROGRAM` header.

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. OUTER-PGM.
       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-DATA         PIC X(10).
       PROCEDURE DIVISION.
           CALL 'INNER-PGM' USING WS-DATA
           GOBACK
           .

       IDENTIFICATION DIVISION.
       PROGRAM-ID. INNER-PGM.
       DATA DIVISION.
       LINKAGE SECTION.
       01  LS-DATA          PIC X(10).
       PROCEDURE DIVISION USING LS-DATA.
           MOVE 'HELLO' TO LS-DATA
           GOBACK
           .
       END PROGRAM INNER-PGM.
       END PROGRAM OUTER-PGM.
```

Nested programs have these characteristics:
- They are contained within the source of another program.
- They can only be called by their direct containing program (unless declared `COMMON`).
- They do not share data with the containing program unless data is passed via `USING` or the nested program is declared with the `COMMON` attribute.
- Multiple levels of nesting are permitted.
- Each nested program has its own complete set of divisions.

### COMMON and INITIAL Attributes

These attributes are specified on the `PROGRAM-ID` paragraph and control the visibility and state behavior of nested programs.

**COMMON attribute**: Makes a nested program callable by other programs at the same nesting level within the same containing program, not just by the direct parent.

```cobol
       PROGRAM-ID. UTILITY-PGM IS COMMON.
```

Without `COMMON`, a nested program can only be called by its immediately containing program. With `COMMON`, sibling programs (other programs nested at the same level) can also call it.

**INITIAL attribute**: Causes the program to be returned to its initial state every time it is called. All WORKING-STORAGE items are reset to their VALUE clauses (or to their default initial state), and all internal file statuses are reset. This is equivalent to the program being freshly loaded each time.

```cobol
       PROGRAM-ID. STATELESS-PGM IS INITIAL.
```

Both attributes can be combined:

```cobol
       PROGRAM-ID. SHARED-UTIL IS COMMON INITIAL.
```

---

## Syntax & Examples

### The CALL Statement

The `CALL` statement transfers control from the calling program to a subprogram.

**Basic syntax:**

```cobol
       CALL {literal-1 | identifier-1}
           [USING {[BY REFERENCE] {identifier-2} ...}
                  {BY CONTENT      {identifier-3 | literal-2} ...}
                  {BY VALUE        {identifier-4 | literal-3} ...}]
           [RETURNING identifier-5]
           [ON OVERFLOW imperative-statement-1]
           [ON EXCEPTION imperative-statement-2]
           [NOT ON EXCEPTION imperative-statement-3]
       END-CALL
```

**Simple call with no parameters:**

```cobol
       CALL 'INITPGM'
       END-CALL
```

**Call with parameters:**

```cobol
       CALL 'CALCPGM' USING WS-INPUT-REC
                             WS-OUTPUT-REC
                             WS-RETURN-CODE
       END-CALL
```

**Call with exception handling:**

```cobol
       CALL WS-PGM-NAME USING WS-PARM1
           ON EXCEPTION
               DISPLAY 'PROGRAM NOT FOUND: ' WS-PGM-NAME
               MOVE 'Y' TO WS-ERROR-FLAG
           NOT ON EXCEPTION
               MOVE 'N' TO WS-ERROR-FLAG
       END-CALL
```

The `ON EXCEPTION` (or `ON OVERFLOW` in older COBOL standards) clause fires when the called program cannot be found or loaded. This is particularly important for dynamic calls where the program name might be invalid or the load module might not be available.

### BY REFERENCE, BY CONTENT, BY VALUE

These phrases control how data is passed from the calling program to the called program. They are specified in the `USING` clause of the `CALL` statement.

**BY REFERENCE** (the default): The address of the data item in the calling program is passed. The called program operates directly on the caller's data. Any changes made by the subprogram are immediately visible to the calling program.

```cobol
      * These two statements are equivalent:
       CALL 'SUBPGM' USING BY REFERENCE WS-RECORD
       CALL 'SUBPGM' USING WS-RECORD
```

**BY CONTENT**: A copy of the data is made, and the address of the copy is passed. The called program can modify the copy, but the original data in the calling program is not affected. This protects the caller's data while still allowing the subprogram to use standard LINKAGE SECTION references.

```cobol
       CALL 'SUBPGM' USING BY CONTENT WS-ACCOUNT-NO
                            BY CONTENT WS-AMOUNT
```

You can also pass literals BY CONTENT:

```cobol
       CALL 'SUBPGM' USING BY CONTENT 'INQUIRY'
                            BY CONTENT 12345
```

**BY VALUE**: The actual value is passed (not an address). This is primarily used when calling non-COBOL programs (such as C functions or system APIs) that expect parameters passed by value on the stack or in registers.

```cobol
       CALL 'C-FUNCTION' USING BY VALUE WS-INT-PARM
                                BY VALUE WS-LENGTH
```

**Mixing passing modes in a single CALL:**

```cobol
       CALL 'SUBPGM' USING BY REFERENCE WS-INPUT-REC
                            BY CONTENT   WS-CONTROL-FLAG
                            BY REFERENCE WS-OUTPUT-REC
```

The passing mode keyword applies to all subsequent parameters until a different mode keyword appears:

```cobol
      * WS-A and WS-B are BY REFERENCE, WS-C is BY CONTENT
       CALL 'SUBPGM' USING BY REFERENCE WS-A
                                        WS-B
                            BY CONTENT   WS-C
```

**Summary of passing modes:**

| Mode | Address or Value? | Caller's Data Modified? | Literals Allowed? | Primary Use |
|---|---|---|---|---|
| BY REFERENCE | Address of original | Yes | No | Standard COBOL-to-COBOL calls |
| BY CONTENT | Address of copy | No | Yes | Protecting caller's data |
| BY VALUE | Actual value | No (no address to modify) | Yes | Calling C/system APIs |

### LINKAGE SECTION

The LINKAGE SECTION is coded in the DATA DIVISION of the called program. It describes the data items that the program expects to receive from its caller. Items in the LINKAGE SECTION do not have storage allocated by the called program itself; instead, they map onto the memory passed by the calling program.

```cobol
       DATA DIVISION.
       LINKAGE SECTION.
       01  LS-INPUT-RECORD.
           05  LS-ACCOUNT-NO       PIC X(10).
           05  LS-ACCOUNT-NAME     PIC X(30).
           05  LS-BALANCE          PIC S9(9)V99 COMP-3.
       01  LS-OUTPUT-RECORD.
           05  LS-RESULT-CODE      PIC X(02).
           05  LS-MESSAGE          PIC X(80).
       01  LS-RETURN-CODE          PIC S9(04) COMP.
```

Key rules for LINKAGE SECTION:
- Items in the LINKAGE SECTION are not accessible until the program is called with corresponding `USING` operands (or until addresses are set via `SET ADDRESS OF`).
- VALUE clauses are not allowed on items in the LINKAGE SECTION, except for level-88 condition names.
- Data descriptions in the LINKAGE SECTION must match (or be compatible with) the data passed by the calling program in terms of size and alignment.
- Only 01-level and 77-level items in the LINKAGE SECTION can appear in the `PROCEDURE DIVISION USING` clause.

### PROCEDURE DIVISION USING

The `USING` clause on the `PROCEDURE DIVISION` header of the called program establishes the correspondence between the caller's arguments and the called program's LINKAGE SECTION items.

```cobol
       PROCEDURE DIVISION USING LS-INPUT-RECORD
                                LS-OUTPUT-RECORD
                                LS-RETURN-CODE.
```

The parameters are matched positionally: the first item in the caller's `USING` corresponds to the first item in the called program's `USING`, and so on.

**With BY REFERENCE / BY VALUE on the receiving side:**

```cobol
       PROCEDURE DIVISION USING BY REFERENCE LS-INPUT-REC
                                BY VALUE     LS-LENGTH.
```

When `BY VALUE` is specified on the `PROCEDURE DIVISION USING`, the called program receives the actual value rather than an address. This must match the corresponding `BY VALUE` in the caller's `CALL` statement.

**With RETURNING:**

```cobol
       PROCEDURE DIVISION USING LS-INPUT-REC
                          RETURNING LS-RESULT.
```

The `RETURNING` item is an additional LINKAGE SECTION item whose value is made available to the caller after the subprogram returns. The caller accesses it via the `RETURNING` phrase on the `CALL` statement or via the `RETURN-CODE` special register (depending on the implementation).

### CANCEL Statement

The `CANCEL` statement releases the resources associated with a dynamically called subprogram and ensures that the next `CALL` to that program will find it in its initial state.

```cobol
       CANCEL 'SUBPGM1'
       CANCEL WS-PGM-NAME
```

After a `CANCEL`:
- The subprogram's memory may be freed (implementation-dependent).
- The next `CALL` to that program will re-initialize it as if it had never been called: WORKING-STORAGE is reset to VALUE clauses, file statuses are reset, and PERFORM stacks are cleared.
- If the program was statically linked, `CANCEL` has no effect on memory but still resets the program's state on some implementations.

**Multiple programs can be cancelled in one statement:**

```cobol
       CANCEL 'SUBPGM1' 'SUBPGM2' 'SUBPGM3'
```

**Important**: You must not `CANCEL` a program that is still active (i.e., one that has called your program or is higher in the current call chain). Doing so produces undefined behavior.

### RETURN-CODE Special Register

`RETURN-CODE` is a special register (typically `PIC S9(4) COMP` or `PIC S9(4) BINARY`) that provides a communication channel between calling and called programs.

**In the called program** -- set before returning via `GOBACK`:

```cobol
       PROCEDURE DIVISION.
           PERFORM PROCESS-LOGIC
           IF WS-SUCCESS
               MOVE 0 TO RETURN-CODE
           ELSE
               MOVE 16 TO RETURN-CODE
           END-IF
           GOBACK
           .
```

**In the calling program** -- check after the call:

```cobol
       CALL 'SUBPGM' USING WS-INPUT-REC
       END-CALL
       EVALUATE RETURN-CODE
           WHEN 0
               CONTINUE
           WHEN OTHER
               PERFORM ERROR-ROUTINE
       END-EVALUATE
```

The `RETURN-CODE` register is automatically set by COBOL subprograms. Note that if the `RETURNING` phrase is used on the `CALL` statement, the returned value goes into the specified identifier, not into `RETURN-CODE`.

See [error_handling.md](error_handling.md) for the complete RETURN-CODE reference, including conventional values (0, 4, 8, 12, 16) and accumulator patterns.

### ENTRY Statement

The `ENTRY` statement defines an alternate entry point into a subprogram. This allows a single subprogram to be called by different names, potentially with different parameter lists at each entry point.

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. MULTIPGM.
       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-INTERNAL-DATA   PIC X(100).
       LINKAGE SECTION.
       01  LS-FULL-REC        PIC X(200).
       01  LS-KEY-ONLY        PIC X(10).
       01  LS-RESULT          PIC X(80).

       PROCEDURE DIVISION.
      * This entry point is never normally reached via CALL 'MULTIPGM'
      * because alternate entries are used instead.
           GOBACK
           .

       ENTRY 'MPGM-INIT' USING LS-FULL-REC.
           PERFORM INITIALIZE-LOGIC
           GOBACK
           .

       ENTRY 'MPGM-QUERY' USING LS-KEY-ONLY LS-RESULT.
           PERFORM QUERY-LOGIC
           GOBACK
           .

       ENTRY 'MPGM-CLOSE'.
           PERFORM CLEANUP-LOGIC
           GOBACK
           .
```

**Calling the alternate entry points:**

```cobol
       CALL 'MPGM-INIT' USING WS-FULL-REC
       CALL 'MPGM-QUERY' USING WS-KEY WS-RESULT
       CALL 'MPGM-CLOSE'
```

Key points about `ENTRY`:
- The program maintains its state across calls to different entry points (WORKING-STORAGE is not reset).
- Each entry point can have a different `USING` parameter list.
- `CANCEL` of any entry name cancels the entire program.
- The `ENTRY` statement is widely supported but is considered somewhat legacy. Modern practice often favors separate subprograms or an explicit operation-code parameter instead.

### Calling Conventions

Calling conventions determine how parameters are physically passed at the machine level (on the stack, via registers, in a parameter list with addresses, etc.). This matters primarily when COBOL programs call or are called by non-COBOL programs.

**COBOL-to-COBOL calls**: The default calling convention is used. On most mainframe and COBOL implementations, parameters are passed via a list of addresses (pointers). The called program's LINKAGE SECTION items are mapped to these addresses. No special syntax is needed.

**COBOL-to-C (or other languages)**: Use `BY VALUE` to pass parameters by value, as C functions typically expect. You may also need compiler directives or the `CALL-CONVENTION` clause.

```cobol
       CALL 'myFunction' USING BY VALUE WS-INT-PARM
                                BY REFERENCE WS-BUFFER
                                BY VALUE WS-BUFFER-LEN
       END-CALL
```

**Returning control from a subprogram:**

`GOBACK` is the preferred way to return from a subprogram because it returns control to the caller without terminating the entire run unit. `STOP RUN` in a subprogram terminates all programs in the call chain, which is almost always undesirable. See [paragraph_flow.md](paragraph_flow.md) for the full comparison of STOP RUN, GOBACK, and EXIT PROGRAM.

---

## Common Patterns

### Standard Subprogram Call with Error Handling

This is the most common pattern seen in production COBOL systems. A calling program invokes a subprogram, passes a communication area, and checks the return status.

```cobol
      *---------------------------------------------------------
      * CALLING PROGRAM
      *---------------------------------------------------------
       WORKING-STORAGE SECTION.
       01  WS-COMM-AREA.
           05  WS-FUNCTION-CODE    PIC X(04).
           05  WS-INPUT-DATA       PIC X(200).
           05  WS-OUTPUT-DATA      PIC X(500).
           05  WS-RESPONSE-CODE    PIC S9(04) COMP.
           05  WS-ERROR-MSG        PIC X(80).
       01  WS-PGM-NAME            PIC X(08) VALUE 'ACCTPGM'.

       PROCEDURE DIVISION.
           INITIALIZE WS-COMM-AREA
           MOVE 'INQY' TO WS-FUNCTION-CODE
           MOVE WS-ACCOUNT-NO TO WS-INPUT-DATA

           CALL WS-PGM-NAME USING WS-COMM-AREA
               ON EXCEPTION
                   MOVE -1 TO WS-RESPONSE-CODE
                   STRING 'PROGRAM ' WS-PGM-NAME ' NOT FOUND'
                       DELIMITED BY SIZE
                       INTO WS-ERROR-MSG
                   END-STRING
           END-CALL

           EVALUATE WS-RESPONSE-CODE
               WHEN 0
                   PERFORM PROCESS-SUCCESS
               WHEN OTHER
                   PERFORM PROCESS-ERROR
           END-EVALUATE
           .
```

### Single Subprogram with Multiple Operations (Dispatcher Pattern)

Instead of using multiple `ENTRY` points, modern practice favors a single entry with an operation code.

```cobol
      *---------------------------------------------------------
      * CALLED SUBPROGRAM -- DISPATCHER PATTERN
      *---------------------------------------------------------
       IDENTIFICATION DIVISION.
       PROGRAM-ID. CUSTPGM.
       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-FIRST-TIME-FLAG     PIC X(01) VALUE 'Y'.
           88  FIRST-TIME                    VALUE 'Y'.
       LINKAGE SECTION.
       01  LS-REQUEST.
           05  LS-OPERATION        PIC X(04).
               88  OP-INIT                   VALUE 'INIT'.
               88  OP-READ                   VALUE 'READ'.
               88  OP-UPDT                   VALUE 'UPDT'.
               88  OP-TERM                   VALUE 'TERM'.
           05  LS-DATA             PIC X(500).
           05  LS-STATUS           PIC S9(04) COMP.

       PROCEDURE DIVISION USING LS-REQUEST.
       0000-MAIN.
           MOVE 0 TO LS-STATUS

           EVALUATE TRUE
               WHEN OP-INIT
                   PERFORM 1000-INITIALIZE
               WHEN OP-READ
                   PERFORM 2000-READ-CUSTOMER
               WHEN OP-UPDT
                   PERFORM 3000-UPDATE-CUSTOMER
               WHEN OP-TERM
                   PERFORM 9000-TERMINATE
               WHEN OTHER
                   MOVE 8 TO LS-STATUS
           END-EVALUATE

           GOBACK
           .
```

### Preserving State Across Calls

A subprogram retains its WORKING-STORAGE values between calls (unless it is declared `INITIAL` or has been `CANCEL`led). This is commonly used for file-handling subprograms that open a file on the first call and keep it open for subsequent calls.

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. FILEPGM.

       DATA DIVISION.
       WORKING-STORAGE SECTION.
       01  WS-FILE-OPEN-FLAG      PIC X(01) VALUE 'N'.
           88  FILE-IS-OPEN                  VALUE 'Y'.
           88  FILE-NOT-OPEN                 VALUE 'N'.
       01  WS-RECORD-COUNT        PIC 9(07) COMP-3 VALUE 0.

       LINKAGE SECTION.
       01  LS-OPERATION            PIC X(04).
       01  LS-RECORD               PIC X(200).
       01  LS-STATUS               PIC S9(04) COMP.

       PROCEDURE DIVISION USING LS-OPERATION
                                LS-RECORD
                                LS-STATUS.
           MOVE 0 TO LS-STATUS

           EVALUATE LS-OPERATION
               WHEN 'OPEN'
                   IF FILE-NOT-OPEN
                       OPEN INPUT CUST-FILE
                       SET FILE-IS-OPEN TO TRUE
                   END-IF
               WHEN 'READ'
                   IF FILE-IS-OPEN
                       READ CUST-FILE INTO LS-RECORD
                       ADD 1 TO WS-RECORD-COUNT
                   ELSE
                       MOVE 8 TO LS-STATUS
                   END-IF
               WHEN 'CLOS'
                   IF FILE-IS-OPEN
                       CLOSE CUST-FILE
                       SET FILE-NOT-OPEN TO TRUE
                   END-IF
           END-EVALUATE

           GOBACK
           .
```

### Chained Calls and Layered Architecture

Production systems commonly layer calls: a main program calls a business-logic subprogram, which calls a data-access subprogram, which calls a utility subprogram.

```cobol
      * MAIN (batch driver) --> BIZLOGIC --> DATAACCS --> DB2UTIL
      *
      * Each layer only knows about the layer directly below it.
      * Data is passed through communication areas at each boundary.

       CALL 'BIZLOGIC' USING WS-BIZ-REQUEST
       END-CALL
       IF WS-BIZ-STATUS NOT = 0
           PERFORM ERROR-HANDLER
       END-IF
```

### Passing Variable-Length Data

When the called program needs to handle variable-length data, the length is typically passed as a separate parameter.

```cobol
      * CALLING PROGRAM
       01  WS-BUFFER              PIC X(5000).
       01  WS-BUFFER-LEN          PIC S9(04) COMP VALUE 5000.

       CALL 'PARSEPGM' USING WS-BUFFER
                              WS-BUFFER-LEN
       END-CALL
```

```cobol
      * CALLED PROGRAM
       LINKAGE SECTION.
       01  LS-BUFFER              PIC X(5000).
       01  LS-BUFFER-LEN          PIC S9(04) COMP.

       PROCEDURE DIVISION USING LS-BUFFER
                                LS-BUFFER-LEN.
      * Only process the first LS-BUFFER-LEN bytes
           MOVE LS-BUFFER(1:LS-BUFFER-LEN) TO WS-WORK-AREA
           ...
```

### Dynamic Program Selection

Selecting which subprogram to call at runtime based on data values.

```cobol
       01  WS-PGM-TABLE.
           05  FILLER             PIC X(12) VALUE 'CHKGPGM CHKG'.
           05  FILLER             PIC X(12) VALUE 'SAVGPGM SAVG'.
           05  FILLER             PIC X(12) VALUE 'LOANPGM LOAN'.
           05  FILLER             PIC X(12) VALUE 'CDPGM   CD  '.
       01  WS-PGM-ENTRIES REDEFINES WS-PGM-TABLE.
           05  WS-PGM-ENTRY       OCCURS 4 TIMES.
               10  WS-PGM-NAME   PIC X(08).
               10  WS-PGM-CODE   PIC X(04).
       01  WS-IDX                 PIC 9(02) COMP.

       PROCEDURE DIVISION.
           PERFORM VARYING WS-IDX FROM 1 BY 1
               UNTIL WS-IDX > 4
               IF WS-PGM-CODE(WS-IDX) = WS-ACCT-TYPE
                   CALL WS-PGM-NAME(WS-IDX) USING WS-ACCT-REC
                   END-CALL
               END-IF
           END-PERFORM
```

---

## Gotchas

- **Parameter count mismatch**: If the calling program passes fewer parameters than the called program expects in its `PROCEDURE DIVISION USING`, the called program will reference memory beyond what was provided, causing unpredictable behavior or abends. COBOL does not validate parameter counts at runtime in most implementations. Always ensure the number and order of parameters match exactly.

- **Parameter size mismatch**: The LINKAGE SECTION description in the called program must not describe a data item larger than what the calling program actually passes. If the called program's LINKAGE SECTION declares a 200-byte record but the caller passes a 100-byte field, the called program will read or write beyond the caller's data -- corrupting memory silently.

- **BY REFERENCE is the default**: If you omit the passing mode keyword, `BY REFERENCE` is assumed. This means the called program can (and often will) modify the caller's data. If you need to protect data, explicitly use `BY CONTENT`.

- **CANCEL of an active program**: Cancelling a program that is currently in the call chain (i.e., it has called your program, or it is somewhere above you in the call stack) causes undefined behavior. Only cancel programs that have returned control and are no longer active.

- **STOP RUN in a subprogram**: Using `STOP RUN` in a subprogram terminates the entire run unit, not just the subprogram. This abruptly ends the calling program and all programs in the chain. Use `GOBACK` instead.

- **Stale state after repeated calls**: Because a subprogram retains its WORKING-STORAGE values between calls, data from a previous call can bleed into subsequent calls if you do not properly initialize. This is a frequent source of production bugs, especially in batch programs that call a subprogram thousands of times in a loop. Always initialize relevant fields at the start of each call, or declare the program `INITIAL`.

- **Dynamic call program-not-found**: If a dynamically called program cannot be found, the behavior depends on whether you coded `ON EXCEPTION`. Without it, the program abends. Always include `ON EXCEPTION` handling for dynamic calls where the program name comes from data.

- **ENTRY statement and CANCEL interaction**: If a program has multiple ENTRY points (e.g., `ENTRY 'PGM-INIT'` and `ENTRY 'PGM-PROC'`), cancelling by any entry name cancels the entire program. This can be surprising if you expected to cancel only one entry point's resources.

- **ADDRESS OF linkage items before a CALL**: LINKAGE SECTION items have no valid address until the program is called with corresponding `USING` parameters. Referencing them in declaratives or initialization code that runs before the `USING` parameters are established will cause abends or corruption.

- **Mixing BY VALUE with BY REFERENCE on the receiving side**: If the `CALL` passes `BY VALUE` but the called program's `PROCEDURE DIVISION USING` says `BY REFERENCE` (or vice versa), the called program will misinterpret the parameter. The caller and callee must agree on the passing mode.

- **RETURN-CODE is global and volatile**: The `RETURN-CODE` special register is overwritten by every `CALL` that returns. If you call two subprograms in sequence and only check `RETURN-CODE` after the second call, you lose the first program's return code. Save it immediately after each `CALL` if you need it later.

- **Nested program visibility**: A nested program that is not declared `COMMON` is invisible to sibling nested programs. If program B and program C are both nested inside program A, B cannot call C (and C cannot call B) unless C is declared `COMMON`. This restriction catches many developers off guard.

- **INITIAL programs and file handling**: An `INITIAL` program resets all state on every call. If the program opens a file, the open status is lost on the next call. Avoid using `INITIAL` on programs that need to maintain file state or accumulated counters across calls.

- **Recursive calls**: Standard COBOL does not support recursion by default. A program calling itself (directly or indirectly) causes undefined behavior unless the `RECURSIVE` attribute is specified on the PROGRAM-ID. Even with `RECURSIVE`, careful use of LOCAL-STORAGE (rather than WORKING-STORAGE) is required, since WORKING-STORAGE is shared across all invocations.

---

## Related Topics

- **cobol_structure.md** -- Covers the four divisions of a COBOL program (IDENTIFICATION, ENVIRONMENT, DATA, PROCEDURE). Subprograms follow the same divisional structure; the LINKAGE SECTION is part of the DATA DIVISION.
- **working_storage.md** -- Details WORKING-STORAGE SECTION and LOCAL-STORAGE SECTION. Understanding these is essential because WORKING-STORAGE retains values across subprogram calls while LOCAL-STORAGE is allocated fresh per invocation.
- **paragraph_flow.md** -- Covers PERFORM, GO TO, and control flow within a program. Subprogram calls via CALL transfer control between programs, while PERFORM transfers control within a program. Understanding both is necessary for tracing execution paths.
- **cics.md** -- In CICS environments, subprogram calls work differently. CICS uses EXEC CICS LINK and EXEC CICS XCTL instead of (or in addition to) the COBOL CALL statement. Programs share data via the COMMAREA or CHANNEL/CONTAINER mechanisms rather than USING parameters.
- **data_types.md** -- Covers PIC clauses, COMP/COMP-3 formats, and data representation. Parameter passing between programs requires matching data types and sizes across the CALL USING and LINKAGE SECTION definitions.
