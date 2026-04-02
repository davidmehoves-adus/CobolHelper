# CICS

## Description
CICS (Customer Information Control System) is IBM's online transaction processing (OLTP) subsystem for z/OS mainframes. COBOL programs interact with CICS through the EXEC CICS command-level interface, which provides facilities for screen I/O via BMS maps, inter-program communication, temporary and transient data storage, task control, and error handling. This file covers the COBOL-specific aspects of writing CICS application programs — the command syntax, the pseudo-conversational programming model, and the most commonly used CICS services.

## Table of Contents
- [Core Concepts](#core-concepts)
  - [The EXEC CICS / END-EXEC Interface](#the-exec-cics--end-exec-interface)
  - [The Execute Interface Block (EIB)](#the-execute-interface-block-eib)
  - [Pseudo-Conversational Programming Model](#pseudo-conversational-programming-model)
  - [BMS Maps and Mapsets](#bms-maps-and-mapsets)
  - [Communication Area (COMMAREA)](#communication-area-commarea)
  - [Channels and Containers](#channels-and-containers)
  - [Inter-Program Communication](#inter-program-communication)
  - [START and RETRIEVE](#start-and-retrieve)
  - [Temporary Storage Queues (TSQ)](#temporary-storage-queues-tsq)
  - [Transient Data Queues (TDQ)](#transient-data-queues-tdq)
  - [Error Handling](#error-handling)
- [Syntax & Examples](#syntax--examples)
- [Common Patterns](#common-patterns)
- [Gotchas](#gotchas)
- [Related Topics](#related-topics)

## Core Concepts

### The EXEC CICS / END-EXEC Interface

All CICS services are invoked from COBOL using the command-level interface. Every CICS command is delimited by `EXEC CICS` and `END-EXEC`:

```
EXEC CICS <command> <options>
END-EXEC
```

The CICS translator (or the integrated CICS translator in modern compilers) converts these commands into standard COBOL CALL statements before compilation. The translator generates a copybook of working storage fields and inserts the appropriate runtime calls.

Key rules for EXEC CICS commands in COBOL:

- Commands can span multiple lines but must stay within columns 12-72 (standard COBOL margins).
- COBOL literals, data names, and CICS keywords are used within the command block.
- You cannot embed COBOL statements (MOVE, IF, etc.) inside an EXEC CICS block.
- Arguments to CICS options are typically COBOL data names or literals.

### The Execute Interface Block (EIB)

Every CICS program has automatic access to the Execute Interface Block (EIB), a CICS-managed control block containing information about the current task and the most recent CICS command. The EIB is addressed via `DFHEIBLK` in the LINKAGE SECTION.

Key EIB fields:

| Field        | PIC           | Description                                      |
|-------------|---------------|--------------------------------------------------|
| EIBTIME     | S9(7) COMP-3  | Time the task started (0HHMMSS packed decimal)   |
| EIBDATE     | S9(7) COMP-3  | Date the task started (0CYYDDD packed decimal)   |
| EIBTRNID    | X(4)          | Transaction ID that started the task             |
| EIBTASKN    | S9(7) COMP-3  | Task number                                      |
| EIBCPOSN    | S9(4) COMP    | Cursor position from last RECEIVE MAP            |
| EIBCALEN    | S9(4) COMP    | Length of the COMMAREA passed to this program     |
| EIBAID      | X(1)          | Attention identifier (which AID key was pressed) |
| EIBRESP     | S9(8) COMP    | Primary response code from last CICS command     |
| EIBRESP2    | S9(8) COMP    | Secondary response code (additional detail)      |

EIBCALEN is critical for pseudo-conversational logic: it is zero on the first invocation of a transaction and non-zero when the program is re-entered with a COMMAREA.

### Pseudo-Conversational Programming Model

CICS programs use a pseudo-conversational design to avoid holding resources while waiting for user input. The model works as follows:

1. The program sends a screen (map) to the terminal.
2. The program issues `EXEC CICS RETURN TRANSID(xxxx) COMMAREA(...)` to return control to CICS and specify which transaction to invoke when the user responds.
3. CICS releases the task and all associated resources.
4. When the user presses an AID key (Enter, PF key, etc.), CICS starts a new task, invoking the specified transaction.
5. The new program instance receives the COMMAREA and can determine where in the conversation it left off.

This contrasts with conversational programming, where the task remains active during user think time (wasting resources). Pseudo-conversational is the standard and expected approach in virtually all production CICS applications.

The program must distinguish between its first invocation (EIBCALEN = 0) and subsequent invocations (EIBCALEN > 0). This is the fundamental control flow decision in every pseudo-conversational program.

### BMS Maps and Mapsets

Basic Mapping Support (BMS) provides device-independent screen I/O for 3270-type terminals. BMS separates the physical screen layout from the application logic.

**Key terminology:**

- **Mapset** — A load module containing one or more maps. Defined via BMS macro assembler source. The mapset name is used in the SEND MAP and RECEIVE MAP commands.
- **Map** — A single screen layout within a mapset. Each map defines the fields, their positions, attributes, and lengths.
- **Physical map** — The load module that CICS uses to format 3270 data streams. Created by assembling the BMS macros.
- **Symbolic map** — A COBOL copybook generated from the BMS macros. It defines the data structure (DSECT) that the COBOL program uses to populate or read screen fields.

A symbolic map copybook typically defines two structures for each map:

- An **input map** (suffixed with `I`) — used when receiving data from the terminal.
- An **output map** (suffixed with `O`) — used when sending data to the terminal.

Each field in the symbolic map has several associated subfields:

- `xxL` — Length of the data entered (PIC S9(4) COMP).
- `xxF` — Flag byte (attribute indicators).
- `xxA` — Attribute byte (for output, allows modifying field attributes dynamically).
- `xxI` / `xxO` — The actual data content for input or output.

Because the input and output areas typically overlay the same storage (via REDEFINES), you can use a single COPY statement for both directions.

### Communication Area (COMMAREA)

The COMMAREA is a contiguous block of storage used to pass data between pseudo-conversational iterations of a program, or between programs via LINK or XCTL.

Key characteristics:

- Maximum size is 32,763 bytes.
- Defined in the LINKAGE SECTION of the receiving program as `DFHCOMMAREA`.
- The length passed is available in `EIBCALEN`.
- For pseudo-conversational use, the program typically defines a working storage area that mirrors the COMMAREA layout and copies data between them.
- On the first invocation of a transaction (EIBCALEN = 0), the COMMAREA content is undefined and must not be referenced.
- CICS copies the COMMAREA data between task boundaries — the new task gets a fresh copy.

### Channels and Containers

Channels and containers are the modern replacement for COMMAREA, overcoming the 32 KB size limitation.

- A **channel** is a named collection of containers, passed between programs or across transaction boundaries.
- A **container** is a named block of data within a channel. Each container can hold up to 2 GB of data.
- Containers can hold either character data (CHAR) or binary data (BIT).

Programs use `PUT CONTAINER` to store data and `GET CONTAINER` to retrieve it. A channel is passed to another program via the `CHANNEL` option on LINK, XCTL, RETURN, or START.

Advantages over COMMAREA:

- No 32 KB size limit.
- Multiple named data blocks instead of one monolithic area.
- Self-describing — the receiving program can browse available containers.
- Better for service-oriented architectures and web services.

### Inter-Program Communication

CICS provides three primary commands for transferring control between programs:

**LINK** — Calls a subprogram, expecting control to return. Analogous to a COBOL CALL:
- The linked-to program receives control at its beginning.
- When the linked-to program issues RETURN, control passes back to the statement following the LINK.
- Data is passed via COMMAREA or CHANNEL.
- A new program-level logical unit of work is not created; the linked program shares the same task.

**XCTL (Transfer Control)** — Transfers control to another program without expecting return. Analogous to a COBOL GO TO between programs:
- The issuing program is released from storage.
- The receiving program cannot return to the issuing program.
- Data is passed via COMMAREA or CHANNEL.
- Useful for menu-to-function-program navigation.

**RETURN** — Returns control to the invoking program or to CICS:
- If issued in a LINKed program, control returns to the caller.
- If issued at the highest level, control returns to CICS.
- The TRANSID option specifies which transaction to invoke next (pseudo-conversational).
- The COMMAREA or CHANNEL option passes data to the next transaction iteration.

### START and RETRIEVE

**START** initiates a new task asynchronously. The new task runs independently under its own transaction ID, potentially at a later time or on a different CICS region.

Key options:

- `TRANSID` — The transaction to start.
- `FROM` / `LENGTH` — Data to pass to the new task.
- `CHANNEL` — A channel to pass to the new task.
- `INTERVAL` / `TIME` / `AFTER` / `AT` — When to start the new task.
- `TERMID` — Which terminal to associate with the new task.
- `REQID` — A request identifier for later cancellation via CANCEL.

**RETRIEVE** is used by the started task to obtain the data passed via START:

- `INTO` / `SET` — Where to place the retrieved data.
- `LENGTH` — The length of the data area.

START/RETRIEVE is commonly used for deferred processing, batch-like operations triggered from online transactions, and inter-region communication.

### Temporary Storage Queues (TSQ)

Temporary Storage provides named, scratchpad-like queues for storing data that must survive across task boundaries or be shared between tasks.

Key characteristics:

- Each queue is identified by a 1-16 character name.
- Queues can reside in main storage (MAIN) or auxiliary storage (AUXILIARY, i.e., disk).
- Data is stored as numbered items (records), starting from item 1.
- Items can be read by item number or sequentially.
- Items can be updated (rewritten) by item number.
- Queues persist until explicitly deleted or CICS is recycled (for MAIN queues).

Commands:

- `WRITEQ TS` — Writes a new item to the queue (appends). Returns the item number in NUMITEMS.
- `READQ TS` — Reads an item by number or the next item sequentially.
- `WRITEQ TS REWRITE ITEM(n)` — Rewrites an existing item.
- `DELETEQ TS` — Deletes the entire queue.

Common uses: storing browse lists, multi-screen data, audit trails, inter-task communication scratchpads.

### Transient Data Queues (TDQ)

Transient Data provides sequential, one-time-read queues for logging, triggering, and inter-system communication.

Two types:

**Intrapartition TDQ:**
- Defined in the Destination Control Table (DCT).
- Data is stored on a CICS-managed VSAM dataset.
- Each record can be read only once (destructive read).
- Supports automatic trigger levels — when the number of records reaches a threshold, a specified transaction is automatically started.
- Used for audit logging, asynchronous processing triggers, and internal message routing.

**Extrapartition TDQ:**
- Mapped to a physical sequential dataset (QSAM file) defined in JCL.
- Can be input-only or output-only.
- Used for writing reports, reading input files, and interfacing with batch processes.
- Records are read/written sequentially.

Commands:

- `WRITEQ TD` — Writes a record to the queue.
- `READQ TD` — Reads the next record (destructive for intrapartition).
- `DELETEQ TD` — Purges all records (intrapartition only).

### Error Handling

CICS provides multiple mechanisms for handling errors in COBOL programs.

**RESP / RESP2 (Recommended approach):**

The `RESP` option on any EXEC CICS command stores the primary response code in a user-specified field. `RESP2` provides additional detail. After each command, the program tests the response:

```cobol
EXEC CICS READ FILE('CUSTFILE')
    INTO(WS-CUSTOMER-REC)
    RIDFLD(WS-CUST-KEY)
    RESP(WS-RESP)
    RESP2(WS-RESP2)
END-EXEC

EVALUATE WS-RESP
    WHEN DFHRESP(NORMAL)
        CONTINUE
    WHEN DFHRESP(NOTFND)
        PERFORM NOT-FOUND-ROUTINE
    WHEN OTHER
        PERFORM ERROR-ROUTINE
END-EVALUATE
```

The `DFHRESP` translator directive converts symbolic response names to numeric values. Common values include NORMAL (0), ERROR, NOTFND, DUPREC, LENGERR, INVREQ, PGMIDERR, MAPFAIL, QIDERR, ITEMERR, and DISABLED.

**HANDLE CONDITION (Legacy):**

Establishes a GO TO branch for a specific condition. When the condition occurs on a subsequent command, control transfers to the specified paragraph:

```cobol
EXEC CICS HANDLE CONDITION
    NOTFND(NOT-FOUND-PARA)
    ERROR(GENERAL-ERROR-PARA)
END-EXEC
```

This approach is discouraged in modern CICS programming because it creates unpredictable flow of control similar to GO TO statements, making programs difficult to debug and maintain.

**HANDLE AID (Legacy):**

Routes control based on which AID key the user pressed, typically after a RECEIVE MAP:

```cobol
EXEC CICS HANDLE AID
    PF3(EXIT-PARA)
    PF7(PAGE-BACK-PARA)
    PF8(PAGE-FORWARD-PARA)
    ENTER(PROCESS-PARA)
    ANYKEY(INVALID-KEY-PARA)
END-EXEC
```

Like HANDLE CONDITION, this is considered legacy. Modern programs test EIBAID directly using the `DFHAID` copybook.

**HANDLE ABEND:**

Establishes an abend exit — if the task abends, control transfers to a specified label or program:

```cobol
EXEC CICS HANDLE ABEND
    LABEL(ABEND-ROUTINE)
END-EXEC
```

Or to route to a separate abend-handling program:

```cobol
EXEC CICS HANDLE ABEND
    PROGRAM('ABNDPGM')
END-EXEC
```

**IGNORE CONDITION:**

Tells CICS to suppress a specific condition so the program continues to the next statement without error:

```cobol
EXEC CICS IGNORE CONDITION MAPFAIL
END-EXEC
```

## Syntax & Examples

### Basic Pseudo-Conversational Program Structure

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. CUSTINQ.
      *-------------------------------------------------------
      * Customer inquiry - pseudo-conversational CICS program
      *-------------------------------------------------------
       DATA DIVISION.
       WORKING-STORAGE SECTION.

       01  WS-COMMAREA.
           05  WS-CA-CUST-ID       PIC X(6).
           05  WS-CA-RETURN-FLAG   PIC X(1).
               88  FIRST-TIME      VALUE SPACES.

       01  WS-RESP                 PIC S9(8) COMP.
       01  WS-RESP2                PIC S9(8) COMP.

       COPY CUSTSET.
      * (symbolic map copybook for mapset CUSTSET)

       COPY DFHAID.
      * (AID key constants: DFHENTER, DFHPF3, etc.)

       LINKAGE SECTION.
       01  DFHCOMMAREA.
           05  CA-CUST-ID          PIC X(6).
           05  CA-RETURN-FLAG      PIC X(1).

       PROCEDURE DIVISION.
       0000-MAIN.
           IF EIBCALEN = ZERO
               PERFORM 1000-FIRST-TIME
           ELSE
               MOVE DFHCOMMAREA TO WS-COMMAREA
               PERFORM 2000-PROCESS-INPUT
           END-IF
           PERFORM 9000-RETURN
           GOBACK.

       1000-FIRST-TIME.
           PERFORM 3000-SEND-EMPTY-MAP.

       2000-PROCESS-INPUT.
           EVALUATE EIBAID
               WHEN DFHPF3
                   EXEC CICS RETURN
                   END-EXEC
               WHEN DFHENTER
                   PERFORM 4000-RECEIVE-AND-LOOKUP
               WHEN OTHER
                   PERFORM 5000-SEND-INVALID-KEY
           END-EVALUATE.

       3000-SEND-EMPTY-MAP.
           MOVE LOW-VALUES TO CUSTMAPO
           MOVE 'Enter Customer ID and press ENTER'
               TO MSGO
           EXEC CICS SEND MAP('CUSTMAP')
               MAPSET('CUSTSET')
               FROM(CUSTMAPO)
               ERASE
           END-EXEC.

       4000-RECEIVE-AND-LOOKUP.
           EXEC CICS RECEIVE MAP('CUSTMAP')
               MAPSET('CUSTSET')
               INTO(CUSTMAPI)
               RESP(WS-RESP)
           END-EXEC

           IF WS-RESP = DFHRESP(MAPFAIL)
               PERFORM 5000-SEND-INVALID-KEY
           ELSE
               MOVE CUSTIDI TO WS-CA-CUST-ID
               PERFORM 6000-READ-CUSTOMER
           END-IF.

       5000-SEND-INVALID-KEY.
           MOVE LOW-VALUES TO CUSTMAPO
           MOVE 'Invalid key or no data entered'
               TO MSGO
           EXEC CICS SEND MAP('CUSTMAP')
               MAPSET('CUSTSET')
               FROM(CUSTMAPO)
               DATAONLY
           END-EXEC.

       6000-READ-CUSTOMER.
           EXEC CICS READ FILE('CUSTFILE')
               INTO(WS-CUST-RECORD)
               RIDFLD(WS-CA-CUST-ID)
               LENGTH(WS-CUST-LEN)
               RESP(WS-RESP)
           END-EXEC

           EVALUATE WS-RESP
               WHEN DFHRESP(NORMAL)
                   PERFORM 7000-DISPLAY-CUSTOMER
               WHEN DFHRESP(NOTFND)
                   MOVE LOW-VALUES TO CUSTMAPO
                   MOVE 'Customer not found' TO MSGO
                   EXEC CICS SEND MAP('CUSTMAP')
                       MAPSET('CUSTSET')
                       FROM(CUSTMAPO)
                       DATAONLY
                   END-EXEC
               WHEN OTHER
                   PERFORM 9999-ABEND
           END-EVALUATE.

       7000-DISPLAY-CUSTOMER.
           MOVE LOW-VALUES TO CUSTMAPO
           MOVE WS-CUST-NAME   TO CNAMEO
           MOVE WS-CUST-ADDR   TO CADDRO
           MOVE WS-CUST-PHONE  TO CPHONEO
           EXEC CICS SEND MAP('CUSTMAP')
               MAPSET('CUSTSET')
               FROM(CUSTMAPO)
               DATAONLY
           END-EXEC.

       9000-RETURN.
           EXEC CICS RETURN
               TRANSID('CINQ')
               COMMAREA(WS-COMMAREA)
               LENGTH(LENGTH OF WS-COMMAREA)
           END-EXEC.

       9999-ABEND.
           EXEC CICS ABEND ABCODE('CUST')
           END-EXEC.
```

### LINK with COMMAREA

```cobol
      * Calling program
       01  WS-LINK-AREA.
           05  WS-LNK-REQUEST     PIC X(1).
           05  WS-LNK-CUST-ID     PIC X(6).
           05  WS-LNK-CUST-NAME   PIC X(30).
           05  WS-LNK-RETURN-CODE PIC 9(2).

       MOVE 'R' TO WS-LNK-REQUEST
       MOVE '123456' TO WS-LNK-CUST-ID

       EXEC CICS LINK
           PROGRAM('CUSTSRV')
           COMMAREA(WS-LINK-AREA)
           LENGTH(LENGTH OF WS-LINK-AREA)
           RESP(WS-RESP)
       END-EXEC

       IF WS-RESP = DFHRESP(NORMAL)
           IF WS-LNK-RETURN-CODE = 0
               DISPLAY 'Customer: ' WS-LNK-CUST-NAME
           END-IF
       END-IF
```

### XCTL (Transfer Control)

```cobol
       EXEC CICS XCTL
           PROGRAM('MENUPGM')
           COMMAREA(WS-COMMAREA)
           LENGTH(LENGTH OF WS-COMMAREA)
       END-EXEC
```

Note: any code after XCTL in the issuing program will never execute.

### Channels and Containers

```cobol
      * Sending program - populate a channel with containers
       EXEC CICS PUT CONTAINER('CUST-DATA')
           CHANNEL('CUST-CHAN')
           FROM(WS-CUSTOMER-REC)
           FLENGTH(LENGTH OF WS-CUSTOMER-REC)
           CHAR
       END-EXEC

       EXEC CICS PUT CONTAINER('ORDER-LIST')
           CHANNEL('CUST-CHAN')
           FROM(WS-ORDER-TABLE)
           FLENGTH(WS-ORDER-LEN)
           BIT
       END-EXEC

       EXEC CICS LINK
           PROGRAM('ORDERPGM')
           CHANNEL('CUST-CHAN')
           RESP(WS-RESP)
       END-EXEC

      * Receiving program - retrieve containers from current channel
       EXEC CICS GET CONTAINER('CUST-DATA')
           CHANNEL('CUST-CHAN')
           INTO(WS-CUSTOMER-REC)
           FLENGTH(WS-DATA-LEN)
           RESP(WS-RESP)
       END-EXEC
```

### Temporary Storage Queue Operations

```cobol
      * Write a record to a TSQ
       EXEC CICS WRITEQ TS
           QUEUE(WS-QUEUE-NAME)
           FROM(WS-QUEUE-RECORD)
           LENGTH(LENGTH OF WS-QUEUE-RECORD)
           RESP(WS-RESP)
       END-EXEC

      * Read item 3 from a TSQ
       MOVE 3 TO WS-ITEM-NUM
       EXEC CICS READQ TS
           QUEUE(WS-QUEUE-NAME)
           INTO(WS-QUEUE-RECORD)
           LENGTH(WS-REC-LEN)
           ITEM(WS-ITEM-NUM)
           RESP(WS-RESP)
       END-EXEC

      * Rewrite item 3
       EXEC CICS WRITEQ TS
           QUEUE(WS-QUEUE-NAME)
           FROM(WS-QUEUE-RECORD)
           LENGTH(LENGTH OF WS-QUEUE-RECORD)
           ITEM(WS-ITEM-NUM)
           REWRITE
           RESP(WS-RESP)
       END-EXEC

      * Delete the entire queue
       EXEC CICS DELETEQ TS
           QUEUE(WS-QUEUE-NAME)
           RESP(WS-RESP)
       END-EXEC
```

### Transient Data Queue Operations

```cobol
      * Write to an intrapartition TDQ (e.g., audit log)
       EXEC CICS WRITEQ TD
           QUEUE('AUDT')
           FROM(WS-AUDIT-RECORD)
           LENGTH(LENGTH OF WS-AUDIT-RECORD)
           RESP(WS-RESP)
       END-EXEC

      * Read from a TDQ (destructive read)
       EXEC CICS READQ TD
           QUEUE('AUDT')
           INTO(WS-AUDIT-RECORD)
           LENGTH(WS-REC-LEN)
           RESP(WS-RESP)
       END-EXEC
```

### START / RETRIEVE

```cobol
      * Start a deferred task to process a batch
       EXEC CICS START
           TRANSID('BPRC')
           FROM(WS-BATCH-PARMS)
           LENGTH(LENGTH OF WS-BATCH-PARMS)
           INTERVAL(003000)
           REQID('BATCH001')
           RESP(WS-RESP)
       END-EXEC

      * In the started program, retrieve the passed data
       EXEC CICS RETRIEVE
           INTO(WS-BATCH-PARMS)
           LENGTH(WS-PARM-LEN)
           RESP(WS-RESP)
       END-EXEC
```

### SEND MAP Options

```cobol
      * Send map with ERASE (full screen redraw)
       EXEC CICS SEND MAP('MYMAP')
           MAPSET('MYMAPST')
           FROM(MYMAPO)
           ERASE
       END-EXEC

      * Send only data (no redraw of static fields)
       EXEC CICS SEND MAP('MYMAP')
           MAPSET('MYMAPST')
           FROM(MYMAPO)
           DATAONLY
       END-EXEC

      * Send only the map layout with no variable data
       EXEC CICS SEND MAP('MYMAP')
           MAPSET('MYMAPST')
           MAPONLY
           ERASE
       END-EXEC

      * Position cursor at a specific field
       EXEC CICS SEND MAP('MYMAP')
           MAPSET('MYMAPST')
           FROM(MYMAPO)
           DATAONLY
           CURSOR(0)
       END-EXEC
```

When `CURSOR(0)` is specified, CICS positions the cursor at the first field whose length subfield (xxL) is set to -1.

### RESP/RESP2 Error Checking Pattern

```cobol
       01  WS-RESP      PIC S9(8) COMP.
       01  WS-RESP2     PIC S9(8) COMP.

       EXEC CICS WRITEQ TS
           QUEUE('MYQUEUE')
           FROM(WS-DATA)
           LENGTH(LENGTH OF WS-DATA)
           RESP(WS-RESP)
           RESP2(WS-RESP2)
       END-EXEC

       EVALUATE WS-RESP
           WHEN DFHRESP(NORMAL)
               CONTINUE
           WHEN DFHRESP(NOSPACE)
               PERFORM HANDLE-NOSPACE
           WHEN DFHRESP(IOERR)
               PERFORM HANDLE-IO-ERROR
           WHEN OTHER
               STRING 'WRITEQ TS failed: RESP='
                   WS-RESP ' RESP2=' WS-RESP2
                   DELIMITED BY SIZE INTO WS-ERROR-MSG
               PERFORM LOG-ERROR
       END-EVALUATE
```

## Common Patterns

### Pseudo-Conversational Control Flow

The most fundamental pattern in CICS programming. Virtually every online CICS COBOL program follows this structure:

```cobol
       0000-MAIN.
           EVALUATE TRUE
               WHEN EIBCALEN = ZERO
                   PERFORM 1000-FIRST-TIME
               WHEN OTHER
                   MOVE DFHCOMMAREA TO WS-COMMAREA
                   EVALUATE WS-CA-PROCESS-FLAG
                       WHEN 'M'
                           PERFORM 2000-PROCESS-MENU
                       WHEN 'D'
                           PERFORM 3000-PROCESS-DETAIL
                       WHEN 'C'
                           PERFORM 4000-PROCESS-CONFIRM
                   END-EVALUATE
           END-EVALUATE

           EXEC CICS RETURN
               TRANSID('MYTR')
               COMMAREA(WS-COMMAREA)
               LENGTH(LENGTH OF WS-COMMAREA)
           END-EXEC.
```

The process flag in the COMMAREA records the current state of the conversation so the program knows which screen it last sent and what input to expect.

### Unique TSQ Naming with Terminal ID

To avoid collisions when multiple users run the same transaction, TSQ names are commonly built using the terminal ID:

```cobol
       01  WS-QUEUE-NAME        PIC X(16).

       STRING 'CUST' EIBTRMID
           DELIMITED BY SIZE
           INTO WS-QUEUE-NAME
```

This produces queue names like `CUSTT001`, `CUSTT002`, etc., which are unique per terminal.

### Browse Pattern (Scrollable List)

Storing a result set in a TSQ and paging through it:

```cobol
      * Build the list into TSQ
       PERFORM UNTIL WS-EOF = 'Y'
           EXEC CICS READNEXT FILE('CUSTFILE')
               INTO(WS-CUST-REC)
               RIDFLD(WS-CUST-KEY)
               RESP(WS-RESP)
           END-EXEC
           IF WS-RESP = DFHRESP(NORMAL)
               ADD 1 TO WS-ITEM-COUNT
               EXEC CICS WRITEQ TS
                   QUEUE(WS-QUEUE-NAME)
                   FROM(WS-CUST-REC)
                   LENGTH(LENGTH OF WS-CUST-REC)
               END-EXEC
           ELSE
               MOVE 'Y' TO WS-EOF
           END-IF
       END-PERFORM

      * Display a page (items N through N+PAGE-SIZE)
       PERFORM VARYING WS-LINE-IDX FROM 1 BY 1
           UNTIL WS-LINE-IDX > WS-PAGE-SIZE
           COMPUTE WS-ITEM-NUM =
               WS-CURRENT-PAGE-START + WS-LINE-IDX - 1
           IF WS-ITEM-NUM > WS-ITEM-COUNT
               EXIT PERFORM
           END-IF
           EXEC CICS READQ TS
               QUEUE(WS-QUEUE-NAME)
               INTO(WS-CUST-REC)
               LENGTH(WS-REC-LEN)
               ITEM(WS-ITEM-NUM)
           END-EXEC
           PERFORM MOVE-DATA-TO-MAP-LINE
       END-PERFORM
```

### LINK-Based Service Program

A reusable business service invoked via LINK, not tied to any terminal:

```cobol
      * Service program CUSTSRV
       LINKAGE SECTION.
       01  DFHCOMMAREA.
           05  CA-REQUEST         PIC X(1).
               88  CA-READ        VALUE 'R'.
               88  CA-UPDATE      VALUE 'U'.
               88  CA-DELETE      VALUE 'D'.
           05  CA-CUST-ID         PIC X(6).
           05  CA-CUST-DATA       PIC X(200).
           05  CA-RETURN-CODE     PIC 9(2).
               88  CA-SUCCESS     VALUE 00.
               88  CA-NOT-FOUND   VALUE 10.
               88  CA-ERROR       VALUE 99.

       PROCEDURE DIVISION.
           EVALUATE TRUE
               WHEN CA-READ
                   PERFORM READ-CUSTOMER
               WHEN CA-UPDATE
                   PERFORM UPDATE-CUSTOMER
               WHEN CA-DELETE
                   PERFORM DELETE-CUSTOMER
               WHEN OTHER
                   SET CA-ERROR TO TRUE
           END-EVALUATE

           EXEC CICS RETURN END-EXEC.
```

### Cleanup on Exit

Deleting TSQs when the user exits the transaction:

```cobol
       8000-CLEANUP-AND-EXIT.
           EXEC CICS DELETEQ TS
               QUEUE(WS-QUEUE-NAME)
               RESP(WS-RESP)
           END-EXEC

           EXEC CICS SEND CONTROL
               ERASE
               FREEKB
           END-EXEC

           EXEC CICS RETURN
           END-EXEC.
```

### Abend Handling with Logging

```cobol
       0000-MAIN.
           EXEC CICS HANDLE ABEND
               LABEL(9900-ABEND-EXIT)
           END-EXEC.
           ...

       9900-ABEND-EXIT.
           EXEC CICS ASSIGN ABCODE(WS-ABEND-CODE)
           END-EXEC

           STRING 'Abend ' WS-ABEND-CODE
               ' in txn ' EIBTRNID
               ' task ' EIBTASKN
               DELIMITED BY SIZE INTO WS-LOG-MSG

           EXEC CICS WRITEQ TD
               QUEUE('CSMT')
               FROM(WS-LOG-MSG)
               LENGTH(LENGTH OF WS-LOG-MSG)
           END-EXEC

           EXEC CICS RETURN END-EXEC.
```

## Gotchas

- **Referencing DFHCOMMAREA when EIBCALEN is zero.** On the first invocation of a transaction, no COMMAREA exists. Any reference to DFHCOMMAREA fields will produce unpredictable results or an ASRA abend (S0C4). Always check `IF EIBCALEN = ZERO` before touching DFHCOMMAREA.

- **Forgetting to initialize the symbolic map to LOW-VALUES before SEND.** If you do not move LOW-VALUES to the output map area before populating it, residual data in the attribute and data subfields will produce garbled screen output or unexpected attribute overrides.

- **Using HANDLE CONDITION / HANDLE AID in new code.** These legacy commands create GO TO-like control flow that makes programs very difficult to debug and maintain. They are also scoped to the program, not the command, so a HANDLE set up early in the program can unexpectedly intercept a condition from a later, unrelated command. Use RESP/RESP2 testing instead.

- **MAPFAIL on RECEIVE MAP.** If the user presses an AID key without modifying any field, CICS raises MAPFAIL. Programs must either handle this condition via RESP or use IGNORE CONDITION MAPFAIL. Failing to handle it causes an AEIM abend.

- **TSQ name collisions.** If multiple users run the same transaction and the TSQ name does not include the terminal ID (EIBTRMID) or some other unique qualifier, users will read and corrupt each other's data. Always include a unique component in TSQ names.

- **Not checking RESP after every CICS command.** Without RESP checking, a failed CICS command will cause the task to abend with the default CICS error handling (which may or may not be what you want). Coding RESP on every command gives you explicit control.

- **COMMAREA size mismatch between programs.** When using LINK or XCTL, if the calling program passes a COMMAREA shorter than what the receiving program expects, fields beyond the actual length will contain garbage. Always use EIBCALEN or LENGTH to verify the actual size.

- **Forgetting DATAONLY on subsequent SEND MAP calls.** Using ERASE on every SEND causes the entire screen to be repainted, producing a visible flicker. Use ERASE only on the first send and DATAONLY for updates to just the variable data.

- **Assuming TDQ reads are non-destructive.** Intrapartition TDQ reads are destructive — once read, the record is gone. If you need to re-read data, use a TSQ instead. This catches many programmers who confuse TSQ and TDQ semantics.

- **CHANNEL option on RETURN for pseudo-conversational.** While supported, channels on pseudo-conversational RETURN require careful management. The channel data is copied, so very large channels consume significant storage. Consider whether the data truly needs to survive the conversation boundary.

- **LENGTH field for RECEIVE MAP must be declared as PIC S9(4) COMP.** Using the wrong picture clause for length fields causes truncation or abends. CICS length fields are halfword binary (PIC S9(4) COMP) unless FLENGTH is used, which expects fullword (PIC S9(8) COMP).

- **Mixing HANDLE CONDITION with RESP.** If both are active, RESP takes precedence and the HANDLE is ignored for that command. However, mixing the two styles in the same program creates confusion and maintenance headaches. Choose one approach and use it consistently.

- **Browsing a file without issuing ENDBR.** STARTBR establishes a browse position and holds a file control block. If the program does not issue ENDBR (e.g., due to an error path), the browse remains active, potentially causing resource shortages. Always ensure ENDBR is executed on all code paths, including error paths.

- **Cursor positioning with symbolic cursor.** When using `CURSOR(0)` on SEND MAP, you must set the length subfield (xxL) of the target field to -1 (`MOVE -1 TO xxL`). If no field has its length set to -1, the cursor goes to position (0,0), which is often an unprotected attribute byte.

## Related Topics

- **[error_handling.md](error_handling.md)** — General COBOL error-handling techniques. CICS RESP/RESP2 checking is an extension of defensive coding practices; HANDLE ABEND complements standard COBOL error strategies.
- **[copybooks.md](copybooks.md)** — BMS symbolic maps are generated as COBOL copybooks included via COPY statements. The DFHAID and DFHBMSCA copybooks are standard CICS includes used in virtually every CICS program.
- **[working_storage.md](working_storage.md)** — CICS programs define their COMMAREA layout, symbolic maps, response fields, and queue data areas in WORKING-STORAGE. Understanding working storage layout is essential for CICS pseudo-conversational design.
- **[subprograms.md](subprograms.md)** — CICS LINK is analogous to COBOL CALL for invoking subprograms. The COMMAREA/CHANNEL passing mechanism in LINK parallels parameter passing in static and dynamic CALL.
- **[db2_embedded_sql.md](db2_embedded_sql.md)** — Many CICS programs also use embedded DB2 SQL. CICS-DB2 programs require both the CICS translator and the DB2 precompiler. Understanding how EXEC CICS and EXEC SQL coexist in the same program is critical for online database applications.
