# Report Writer

## Description

The COBOL Report Writer module is a declarative facility for producing formatted printed reports. Rather than coding explicit WRITE statements, line counters, and page-break logic, the programmer describes the report layout in the DATA DIVISION (using RD and report group entries) and then drives report production with three PROCEDURE DIVISION statements: INITIATE, GENERATE, and TERMINATE. The Report Writer runtime handles page overflow, control breaks, heading and footing generation, and accumulator (SUM counter) arithmetic automatically.

## Table of Contents

- [Core Concepts](#core-concepts)
  - [How Report Writer Works](#how-report-writer-works)
  - [RD (Report Description)](#rd-report-description)
  - [Report Groups](#report-groups)
  - [Report Group Types](#report-group-types)
  - [LINE, COLUMN, and NEXT GROUP Clauses](#line-column-and-next-group-clauses)
  - [PAGE LIMIT Clause](#page-limit-clause)
  - [CONTROL Clause](#control-clause)
  - [SOURCE, SUM, and VALUE Clauses](#source-sum-and-value-clauses)
  - [GROUP INDICATE](#group-indicate)
  - [PROCEDURE DIVISION Statements](#procedure-division-statements)
- [Syntax & Examples](#syntax--examples)
  - [Minimal Report Writer Program](#minimal-report-writer-program)
  - [Control Break Report](#control-break-report)
  - [Multi-Level Control Break with SUMs](#multi-level-control-break-with-sums)
- [Common Patterns](#common-patterns)
  - [Classic Batch Listing Report](#classic-batch-listing-report)
  - [Summary-Only Reports](#summary-only-reports)
  - [Report Writer vs Manual Report Generation](#report-writer-vs-manual-report-generation)
- [Gotchas](#gotchas)
- [Related Topics](#related-topics)

## Core Concepts

### How Report Writer Works

The Report Writer module separates **what** the report looks like from **when** report lines are produced. The programmer defines the layout declaratively in the DATA DIVISION and then, in the PROCEDURE DIVISION, reads input records and issues GENERATE statements. The Report Writer Control System (RWCS) -- the runtime engine -- decides at each GENERATE which report groups to present, performs page-break detection, fires control-break headings and footings, rolls up SUM counters, and writes output lines to the report file.

The lifecycle of a report is always:

1. **INITIATE** -- resets all counters and prepares the report.
2. **GENERATE** (repeated) -- presents a DETAIL group or triggers control-break processing.
3. **TERMINATE** -- produces final control footings and the REPORT FOOTING.

A report file must be opened (OPEN OUTPUT) before INITIATE and closed (CLOSE) after TERMINATE, but the programmer never issues WRITE statements for the report -- the RWCS does that internally.

### RD (Report Description)

The RD entry appears in the REPORT SECTION of the DATA DIVISION. It names the report and specifies global report characteristics. Every report named in a file's SELECT/FD must have a corresponding RD entry.

```
FILE SECTION.
FD  PRINT-FILE
    REPORT IS SALES-REPORT.
```

```
REPORT SECTION.
RD  SALES-REPORT
    CONTROL IS FINAL DEPARTMENT-CODE
    PAGE LIMIT IS 60 LINES
    HEADING 1
    FIRST DETAIL 5
    LAST DETAIL 55
    FOOTING 58.
```

The FD for the report file uses the `REPORT IS` clause instead of a record description. The RD entry may contain:

- **CONTROL IS** -- identifies control break fields (discussed below).
- **PAGE LIMIT** -- defines the logical page geometry.

### Report Groups

A report group is a set of data description entries (level 01 through level 49) subordinate to the RD that together describe one logical "band" of the report -- for example, a page heading, a detail line, or a control footing.

Each report group entry (the 01-level) must include a **TYPE** clause specifying which kind of report group it is. The subordinate entries describe individual print positions using LINE, COLUMN, SOURCE/SUM/VALUE, and PICTURE clauses.

### Report Group Types

There are seven report group types, presented in the order they appear on a page from top to bottom:

| Type | Keyword | When Produced | Typical Use |
|------|---------|---------------|-------------|
| Report Heading | `TYPE IS REPORT HEADING` (or `TYPE RH`) | Once, at INITIATE time (first page only) | Title page, cover page |
| Page Heading | `TYPE IS PAGE HEADING` (or `TYPE PH`) | Top of every page | Column headers, date, page number |
| Control Heading | `TYPE IS CONTROL HEADING identifier` (or `TYPE CH`) | Before the first detail of a new control group | Group header, department name |
| Detail | `TYPE IS DETAIL` (or `TYPE DE`) | Each time GENERATE names this group | Data lines |
| Control Footing | `TYPE IS CONTROL FOOTING identifier` (or `TYPE CF`) | After the last detail of a control group | Subtotals per group |
| Page Footing | `TYPE IS PAGE FOOTING` (or `TYPE PF`) | Bottom of every page | Page number at bottom, confidentiality notice |
| Report Footing | `TYPE IS REPORT FOOTING` (or `TYPE RF`) | Once, at TERMINATE time (last page only) | Grand totals, "End of Report" |

**CONTROL HEADING and CONTROL FOOTING** are tied to a specific control identifier (a data-name) or to the special keyword `FINAL`. There can be multiple CH and CF groups, one per control level.

The order of precedence is: RH, PH, CH (outermost to innermost), DE, CF (innermost to outermost), PF, RF. The RWCS guarantees this ordering automatically.

### LINE, COLUMN, and NEXT GROUP Clauses

These clauses position data on the printed page.

**LINE Clause** -- Specifies which line within the page body a report group entry occupies.

```cobol
01  TYPE IS DETAIL.
    05  LINE PLUS 1.
        10  COLUMN 1    PIC X(20) SOURCE EMPLOYEE-NAME.
        10  COLUMN 25   PIC X(10) SOURCE DEPARTMENT-CODE.
        10  COLUMN 40   PIC ZZ,ZZ9.99 SOURCE SALARY.
```

LINE can be:

- **LINE integer** -- absolute line number within the page.
- **LINE PLUS integer** -- relative to the previous line (most common for detail lines).
- **LINE NEXT PAGE** -- forces a new page before this line prints.

**COLUMN Clause** -- Specifies the horizontal starting position (column number) of a printable item.

```cobol
10  COLUMN 1    PIC X(30) VALUE "EMPLOYEE REPORT".
10  COLUMN 50   PIC X(10) VALUE "PAGE:".
10  COLUMN 61   PIC Z9    SOURCE PAGE-COUNTER.
```

**NEXT GROUP Clause** -- Specifies spacing or page advancement that should occur **after** the report group is presented.

```cobol
01  TYPE IS CONTROL HEADING DEPARTMENT-CODE
    NEXT GROUP PLUS 1.
```

NEXT GROUP can be:

- **NEXT GROUP integer** -- the next group starts at this absolute line.
- **NEXT GROUP PLUS integer** -- skip this many lines after the current group.
- **NEXT GROUP NEXT PAGE** -- force a page break after this group.

### PAGE LIMIT Clause

The PAGE LIMIT clause on the RD entry defines the logical page structure:

```cobol
RD  MY-REPORT
    PAGE LIMIT IS 66 LINES
    HEADING 1
    FIRST DETAIL 6
    LAST DETAIL 58
    FOOTING 62.
```

| Sub-clause | Meaning |
|-----------|---------|
| `PAGE LIMIT IS n LINES` | Total number of lines per logical page |
| `HEADING n` | First line available for RH and PH groups (default: 1) |
| `FIRST DETAIL n` | First line available for CH and DE groups |
| `LAST DETAIL n` | Last line available for DE groups; CH groups can also appear here |
| `FOOTING n` | Last line available for CF and PF groups |

When a GENERATE would cause a DETAIL line to exceed LAST DETAIL, the RWCS automatically triggers page-break processing: it presents the PAGE FOOTING on the current page, advances to a new page, presents the PAGE HEADING, and then continues with the detail line.

### CONTROL Clause

The CONTROL clause on the RD entry defines the hierarchy of control break fields:

```cobol
RD  SALES-REPORT
    CONTROL IS FINAL
               REGION-CODE
               BRANCH-CODE
               SALESPERSON-ID.
```

Fields are listed from **outermost** (most major) to **innermost** (most minor). The special keyword `FINAL` represents the overall report level and is always the outermost control.

At each GENERATE, the RWCS compares the current values of control fields with their previous values. When a change is detected, a **control break** occurs:

1. CONTROL FOOTING groups are presented from innermost to outermost up to (and including) the level that changed.
2. SUM counters are rolled up during footing presentation.
3. CONTROL HEADING groups are then presented from outermost to innermost.
4. The DETAIL line is presented.

This automatic control break mechanism is one of the most powerful features of the Report Writer -- it replaces what can be dozens of lines of procedural save/compare/break logic.

### SOURCE, SUM, and VALUE Clauses

These clauses define the content of printable items within report groups.

**SOURCE Clause** -- The item's value comes from a specified data-name at the time the report group is presented.

```cobol
10  COLUMN 1   PIC X(20) SOURCE EMPLOYEE-NAME.
10  COLUMN 25  PIC 9(5)  SOURCE EMPLOYEE-ID.
```

SOURCE is the most common content clause and works like an implicit MOVE from the named identifier to the print position.

**SUM Clause** -- Defines an accumulator that the RWCS automatically adds to. SUM is valid only in CONTROL FOOTING groups.

```cobol
01  TYPE IS CONTROL FOOTING DEPARTMENT-CODE.
    05  LINE PLUS 2.
        10  COLUMN 1   PIC X(20) VALUE "DEPT TOTAL:".
        10  COLUMN 40  PIC ZZZ,ZZ9.99 SUM SALARY.
```

At each GENERATE, the RWCS adds the current value of SALARY to this SUM counter. When the CONTROL FOOTING is presented (at a control break), the accumulated total is printed and the counter is reset.

**SUM counter rolling** -- A SUM in a higher-level CONTROL FOOTING can name a lower-level SUM counter. This causes automatic roll-up:

```cobol
01  TYPE IS CONTROL FOOTING REGION-CODE.
    05  LINE PLUS 2.
        10  COLUMN 1   PIC X(20) VALUE "REGION TOTAL:".
        10  COLUMN 40  PIC Z,ZZZ,ZZ9.99 SUM DEPT-SALARY-SUM.
```

Here `DEPT-SALARY-SUM` is the data-name assigned to the SUM counter in the department-level CONTROL FOOTING. Each time the department footing is presented, its accumulated value is rolled into the region-level SUM before being reset.

**VALUE Clause** -- The item contains a literal constant.

```cobol
10  COLUMN 1   PIC X(15) VALUE "GRAND TOTAL:".
```

VALUE is used for labels, literals, and static text in headings and footings.

### GROUP INDICATE

The GROUP INDICATE clause causes a printable item to appear only on the first detail line after a control break (or on the first detail line of a new page). On subsequent detail lines within the same control group, the item is suppressed (spaces for alphanumeric, zeros for numeric).

```cobol
01  TYPE IS DETAIL.
    05  LINE PLUS 1.
        10  COLUMN 1   PIC X(20) SOURCE DEPARTMENT-NAME
            GROUP INDICATE.
        10  COLUMN 25  PIC X(30) SOURCE EMPLOYEE-NAME.
        10  COLUMN 60  PIC ZZ,ZZ9.99 SOURCE SALARY.
```

In this example, the department name prints only once at the start of each department group, giving the report a clean, grouped appearance rather than repeating the department name on every line.

### PROCEDURE DIVISION Statements

Three statements drive report production:

**INITIATE** -- Begins report processing. Resets all SUM counters to zero, sets LINE-COUNTER and PAGE-COUNTER to zero, and prepares the RWCS for the first GENERATE.

```cobol
INITIATE SALES-REPORT.
```

The report file must already be open (OPEN OUTPUT) before INITIATE is executed.

**GENERATE** -- Presents a detail report group and triggers any necessary control break processing, page headings, and page footings.

```cobol
GENERATE DETAIL-LINE.
```

GENERATE can also name the RD entry itself (rather than a specific detail group) for summary reporting -- this increments SUM counters without producing a detail line.

```cobol
GENERATE SALES-REPORT.
```

**TERMINATE** -- Ends report processing. Forces final control breaks at all levels (presenting all CONTROL FOOTING groups, including FINAL), presents the REPORT FOOTING, and performs final SUM roll-up.

```cobol
TERMINATE SALES-REPORT.
```

After TERMINATE, the report file should be closed (CLOSE).

**Automatic Counters** -- The RWCS maintains two special registers:

- **LINE-COUNTER** -- current line position on the page (read-only in standard usage).
- **PAGE-COUNTER** -- current page number (can be referenced by SOURCE in PH/PF groups; can be set by the programmer before INITIATE to start at a specific page number).

## Syntax & Examples

### Minimal Report Writer Program

This example shows the smallest complete Report Writer program structure:

```cobol
       IDENTIFICATION DIVISION.
       PROGRAM-ID. MINRPT.

       ENVIRONMENT DIVISION.
       INPUT-OUTPUT SECTION.
       FILE-CONTROL.
           SELECT INPUT-FILE  ASSIGN TO INFILE.
           SELECT REPORT-FILE ASSIGN TO RPTFILE.

       DATA DIVISION.
       FILE SECTION.
       FD  INPUT-FILE.
       01  INPUT-RECORD.
           05  IN-NAME         PIC X(30).
           05  IN-AMOUNT       PIC 9(7)V99.

       FD  REPORT-FILE
           REPORT IS SIMPLE-REPORT.

       REPORT SECTION.
       RD  SIMPLE-REPORT
           PAGE LIMIT IS 60 LINES
           HEADING 1
           FIRST DETAIL 4
           LAST DETAIL 56
           FOOTING 58.

       01  TYPE IS PAGE HEADING.
           05  LINE 1.
               10  COLUMN 1   PIC X(30)
                   VALUE "SIMPLE LISTING REPORT".
               10  COLUMN 50  PIC X(5)
                   VALUE "PAGE:".
               10  COLUMN 56  PIC Z9
                   SOURCE PAGE-COUNTER.
           05  LINE 3.
               10  COLUMN 1   PIC X(30)
                   VALUE "NAME".
               10  COLUMN 35  PIC X(10)
                   VALUE "AMOUNT".

       01  DETAIL-LINE TYPE IS DETAIL.
           05  LINE PLUS 1.
               10  COLUMN 1   PIC X(30)
                   SOURCE IN-NAME.
               10  COLUMN 35  PIC ZZZ,ZZ9.99
                   SOURCE IN-AMOUNT.

       PROCEDURE DIVISION.
       MAIN-LOGIC.
           OPEN INPUT INPUT-FILE
                OUTPUT REPORT-FILE.
           INITIATE SIMPLE-REPORT.

           READ INPUT-FILE
               AT END SET END-OF-FILE TO TRUE
           END-READ.
           PERFORM UNTIL END-OF-FILE
               GENERATE DETAIL-LINE
               READ INPUT-FILE
                   AT END SET END-OF-FILE TO TRUE
               END-READ
           END-PERFORM.

           TERMINATE SIMPLE-REPORT.
           CLOSE INPUT-FILE REPORT-FILE.
           STOP RUN.
```

### Control Break Report

This example demonstrates a single-level control break with subtotals:

```cobol
       REPORT SECTION.
       RD  DEPT-REPORT
           CONTROL IS FINAL DEPT-CODE
           PAGE LIMIT IS 60 LINES
           HEADING 1
           FIRST DETAIL 6
           LAST DETAIL 54
           FOOTING 58.

       01  TYPE IS REPORT HEADING.
           05  LINE 1.
               10  COLUMN 20  PIC X(30)
                   VALUE "DEPARTMENT SALARY REPORT".

       01  TYPE IS PAGE HEADING.
           05  LINE 1.
               10  COLUMN 1   PIC X(10)
                   VALUE "DEPT".
               10  COLUMN 15  PIC X(20)
                   VALUE "EMPLOYEE".
               10  COLUMN 40  PIC X(12)
                   VALUE "SALARY".
               10  COLUMN 60  PIC X(5)
                   VALUE "PAGE".
               10  COLUMN 66  PIC Z9
                   SOURCE PAGE-COUNTER.

       01  DEPT-HEADING TYPE IS CONTROL HEADING DEPT-CODE.
           05  LINE PLUS 2.
               10  COLUMN 1   PIC X(15)
                   VALUE "DEPARTMENT:".
               10  COLUMN 17  PIC X(10)
                   SOURCE DEPT-CODE.

       01  SALARY-LINE TYPE IS DETAIL.
           05  LINE PLUS 1.
               10  COLUMN 15  PIC X(20)
                   SOURCE EMP-NAME.
               10  COLUMN 40  PIC ZZZ,ZZ9.99
                   SOURCE EMP-SALARY.

       01  DEPT-FOOTING TYPE IS CONTROL FOOTING DEPT-CODE.
           05  LINE PLUS 1.
               10  COLUMN 15  PIC X(20)
                   VALUE "DEPARTMENT TOTAL:".
               10  DEPT-SALARY-TOTAL
                   COLUMN 40  PIC Z,ZZZ,ZZ9.99
                   SUM EMP-SALARY.

       01  TYPE IS CONTROL FOOTING FINAL.
           05  LINE PLUS 2.
               10  COLUMN 15  PIC X(20)
                   VALUE "GRAND TOTAL:".
               10  COLUMN 40  PIC Z,ZZZ,ZZ9.99
                   SUM DEPT-SALARY-TOTAL.

       01  TYPE IS PAGE FOOTING.
           05  LINE 58.
               10  COLUMN 25  PIC X(20)
                   VALUE "** CONFIDENTIAL **".
```

### Multi-Level Control Break with SUMs

This pattern shows SUM roll-up across three control levels:

```cobol
       RD  SALES-ANALYSIS
           CONTROL IS FINAL
                      REGION-CODE
                      BRANCH-CODE
           PAGE LIMIT IS 66 LINES
           HEADING 1
           FIRST DETAIL 7
           LAST DETAIL 58
           FOOTING 62.

      * --- Detail line ---
       01  SALE-LINE TYPE IS DETAIL.
           05  LINE PLUS 1.
               10  COLUMN 1   PIC X(10)
                   SOURCE REGION-CODE  GROUP INDICATE.
               10  COLUMN 12  PIC X(10)
                   SOURCE BRANCH-CODE  GROUP INDICATE.
               10  COLUMN 24  PIC X(20)
                   SOURCE CUSTOMER-NAME.
               10  COLUMN 48  PIC ZZZ,ZZ9.99
                   SOURCE SALE-AMOUNT.

      * --- Branch subtotal ---
       01  TYPE IS CONTROL FOOTING BRANCH-CODE.
           05  LINE PLUS 1.
               10  COLUMN 24  PIC X(20)
                   VALUE "BRANCH TOTAL:".
               10  BRANCH-TOTAL
                   COLUMN 48  PIC Z,ZZZ,ZZ9.99
                   SUM SALE-AMOUNT.

      * --- Region subtotal (rolls up branch totals) ---
       01  TYPE IS CONTROL FOOTING REGION-CODE.
           05  LINE PLUS 1.
               10  COLUMN 24  PIC X(20)
                   VALUE "REGION TOTAL:".
               10  REGION-TOTAL
                   COLUMN 48  PIC ZZ,ZZZ,ZZ9.99
                   SUM BRANCH-TOTAL.

      * --- Grand total (rolls up region totals) ---
       01  TYPE IS CONTROL FOOTING FINAL.
           05  LINE PLUS 2.
               10  COLUMN 24  PIC X(20)
                   VALUE "GRAND TOTAL:".
               10  COLUMN 48  PIC ZZZ,ZZZ,ZZ9.99
                   SUM REGION-TOTAL.
```

In this example, SALE-AMOUNT is summed at the branch level. When a branch break occurs, BRANCH-TOTAL is printed and its value is rolled into REGION-TOTAL before being reset. When a region break occurs, REGION-TOTAL rolls into the FINAL-level SUM. At TERMINATE, the grand total reflects the sum of all regions.

## Common Patterns

### Classic Batch Listing Report

The most common Report Writer use case is the batch listing report: read a sequential file, produce a formatted printout with page headings, detail lines, and optional control break subtotals. The PROCEDURE DIVISION is typically very short:

```cobol
       PROCEDURE DIVISION.
       MAIN-PARA.
           OPEN INPUT TRANS-FILE
                OUTPUT PRINT-FILE.
           INITIATE TRANS-REPORT.
           PERFORM READ-AND-GENERATE
               UNTIL END-OF-INPUT.
           TERMINATE TRANS-REPORT.
           CLOSE TRANS-FILE PRINT-FILE.
           STOP RUN.

       READ-AND-GENERATE.
           READ TRANS-FILE INTO WS-TRANSACTION
               AT END
                   SET END-OF-INPUT TO TRUE
                   RETURN
           END-READ.
           GENERATE TRANS-DETAIL.
```

The entire page-break logic, heading reprinting, and overflow detection is handled by the RWCS. In a manual approach, this would require maintaining a line counter, checking for page overflow, performing page-break paragraphs, and resetting counters -- typically 30 to 50 additional lines of procedural code.

### Summary-Only Reports

By issuing `GENERATE report-name` (naming the RD rather than a DETAIL group), the RWCS increments SUM counters without producing detail lines. This is useful for summary-only reports:

```cobol
           PERFORM UNTIL END-OF-INPUT
               READ INPUT-FILE
                   AT END SET END-OF-INPUT TO TRUE
                   NOT AT END
                       GENERATE SUMMARY-REPORT
               END-READ
           END-PERFORM.
           TERMINATE SUMMARY-REPORT.
```

Only the CONTROL HEADING, CONTROL FOOTING, and REPORT HEADING/FOOTING groups are produced. This is an efficient way to produce totals-only reports without writing detail lines.

### Report Writer vs Manual Report Generation

Report Writer and manual report generation each have trade-offs. Understanding when to use each approach is important.

**Advantages of Report Writer:**

- Dramatically less procedural code. A report that needs 200 lines of manual WRITE/line-counter logic may need only 20 lines of PROCEDURE DIVISION code with Report Writer.
- Automatic page overflow handling. No need to maintain and test a line counter.
- Automatic control break processing. Multi-level breaks with SUM roll-up are declared, not coded.
- Easier maintenance. Changing column positions or adding a new control level is a DATA DIVISION change, not a logic change.
- Fewer bugs. The RWCS handles edge cases (page break in the middle of a control break, first/last page handling) that manual code often gets wrong.

**Advantages of Manual Report Generation:**

- More widely understood. Many COBOL programmers have never used Report Writer, so manual code is easier for a team to maintain.
- Full control over output. Complex formatting requirements (conditional lines, variable-format detail lines, graphical elements) are easier with explicit WRITE statements.
- Better compiler support. Some COBOL compilers have limited or no Report Writer support. IBM Enterprise COBOL supports it, but some smaller compilers may not.
- Easier debugging. With manual code, you can set breakpoints on WRITE statements and inspect the output buffer. With Report Writer, the RWCS is a black box.
- No REPORT SECTION overhead. Manual code uses standard FD/01 record descriptions that programmers already understand.

**When to use Report Writer:**

- Batch listing reports with predictable, tabular layouts.
- Reports with multi-level control breaks and subtotals.
- Situations where the PROCEDURE DIVISION should focus on business logic rather than formatting.
- Maintenance-heavy environments where layout changes are frequent.

**When to use manual report generation:**

- Reports with complex conditional formatting.
- Environments where the team is unfamiliar with Report Writer.
- When the target compiler lacks Report Writer support.
- Reports requiring interleaved writes to multiple files or non-sequential output.

In modern practice, many shops have moved report generation to downstream tools (Crystal Reports, BIRT, or ETL-driven report generators), with COBOL producing extract files rather than formatted reports. However, Report Writer remains a valuable tool for batch report programs that run in traditional mainframe environments.

## Gotchas

- **The report file must be OPEN before INITIATE and CLOSE after TERMINATE.** The INITIATE does not open the file, and TERMINATE does not close it. Forgetting this sequence causes runtime errors or empty output. The correct order is always: OPEN, INITIATE, GENERATE (loop), TERMINATE, CLOSE.

- **GENERATE must name a DETAIL group or the RD entry -- nothing else.** You cannot GENERATE a CONTROL HEADING or PAGE HEADING directly. Those groups are produced automatically by the RWCS. Attempting to GENERATE a non-DETAIL group is a compile-time error in most compilers.

- **SUM is only valid in CONTROL FOOTING groups.** Placing a SUM clause in a DETAIL, PAGE HEADING, or any other group type is invalid. If you need a running total displayed on each detail line, you must maintain it manually in WORKING-STORAGE and use SOURCE.

- **Control fields must be listed outermost to innermost on the CONTROL clause.** The order on the CONTROL clause defines the hierarchy. If you list BRANCH before REGION when REGION is the major break, control breaks will fire in the wrong order, producing incorrect subtotals.

- **Input data must be sorted by the control fields in the same order as the CONTROL clause.** Report Writer assumes the input is pre-sorted. If it is not, control breaks will fire on every record (or not fire when expected), producing nonsensical output. Always SORT the input file before feeding it to a Report Writer program, or use the SORT statement with INPUT PROCEDURE/OUTPUT PROCEDURE.

- **PAGE-COUNTER starts at 1 when the first page heading is presented, not at INITIATE time.** If you set PAGE-COUNTER before INITIATE, some implementations may reset it. If you need to start at a specific page number, set PAGE-COUNTER after INITIATE but before the first GENERATE.

- **LINE-COUNTER should be treated as read-only.** While some compilers allow you to modify LINE-COUNTER, doing so can confuse the RWCS page-overflow logic and produce garbled output. Use it only as a SOURCE for display purposes.

- **GROUP INDICATE suppression resets on every new page as well as on control breaks.** This means the indicated item will reappear at the top of each new page even within the same control group, which is usually the desired behavior but can be surprising if you expect suppression to persist across pages.

- **Multiple DETAIL groups are allowed but only one can be named per GENERATE.** If you have multiple DETAIL group types (e.g., DETAIL-LINE-A and DETAIL-LINE-B for different record types), each GENERATE statement must specify which one to produce. The RWCS does not choose automatically.

- **NEXT GROUP NEXT PAGE on a CONTROL FOOTING can cause blank pages.** If a control break coincides with a page break, you may get an unwanted empty page. Test edge cases where control breaks align with page boundaries.

- **Declarative USE BEFORE REPORTING.** The USE BEFORE REPORTING declarative (in the DECLARATIVES section) allows procedural code to execute just before a specific report group is presented. This is the escape hatch for conditional logic (e.g., suppressing a line based on a condition), but overusing it defeats the purpose of the declarative model.

- **Not all compilers support Report Writer identically.** While the feature is part of the COBOL standard, implementation details vary. Features like USE BEFORE REPORTING, PRESENT WHEN (ANS-85 extension), and some SUM behaviors may differ across IBM, Micro Focus, and other compilers. Test on your target platform.

## Related Topics

- **batch_patterns.md** -- Report Writer programs are almost exclusively batch programs. The read-process-write loop, file open/close patterns, and control break concepts discussed in batch patterns directly apply to the PROCEDURE DIVISION of Report Writer programs.
- **file_handling.md** -- The report output file uses standard SELECT/FD entries and OPEN/CLOSE statements. Understanding file handling is essential for setting up the report file correctly and managing input files that feed the report.
- **data_types.md** -- PICTURE clauses in report group entries follow the same rules as regular data items. Understanding numeric editing pictures (Z, *, comma insertion, decimal point) is critical for formatting report columns correctly.
- **working_storage.md** -- Working-storage items are commonly used as SOURCE identifiers, as save areas for control break fields in hybrid approaches, and for flags/switches that control USE BEFORE REPORTING declaratives. Report Writer programs typically still need WORKING-STORAGE for input record processing and program control variables.
