# Conditional Logic

## Description
This file covers all forms of conditional logic in COBOL, including the IF/ELSE/END-IF statement, the EVALUATE/WHEN/END-EVALUATE statement (COBOL's case/switch construct), condition names defined with level 88 entries, and every category of conditional expression recognized by the language: relation conditions, class conditions, sign conditions, combined conditions with AND/OR/NOT, negated conditions, and abbreviated combined relation conditions. Reference this file whenever you need to understand how COBOL programs make decisions or branch execution flow.

## Table of Contents
- [Core Concepts](#core-concepts)
  - [Simple Conditions](#simple-conditions)
  - [Relation Conditions](#relation-conditions)
  - [Class Conditions](#class-conditions)
  - [Sign Conditions](#sign-conditions)
  - [Condition Names (88 Levels)](#condition-names-88-levels)
  - [Combined Conditions (AND / OR / NOT)](#combined-conditions-and--or--not)
  - [Abbreviated Combined Relation Conditions](#abbreviated-combined-relation-conditions)
  - [Negated Conditions](#negated-conditions)
- [Syntax & Examples](#syntax--examples)
  - [IF / ELSE / END-IF](#if--else--end-if)
  - [Nested IF Statements](#nested-if-statements)
  - [EVALUATE / WHEN / END-EVALUATE](#evaluate--when--end-evaluate)
  - [EVALUATE TRUE](#evaluate-true)
  - [EVALUATE with ALSO](#evaluate-with-also)
  - [EVALUATE with THRU](#evaluate-with-thru)
  - [Condition Names in Practice](#condition-names-in-practice)
  - [SET for Condition Names](#set-for-condition-names)
  - [Complex Compound Conditions](#complex-compound-conditions)
- [Common Patterns](#common-patterns)
- [Gotchas](#gotchas)
- [Related Topics](#related-topics)

## Core Concepts

### Simple Conditions

COBOL recognizes several categories of simple conditions. A simple condition is a single test that evaluates to true or false. The simple condition types are:

1. **Relation condition** -- compares two operands with a relational operator.
2. **Class condition** -- tests whether a data item contains a specific class of characters.
3. **Sign condition** -- tests whether a numeric operand is positive, negative, or zero.
4. **Condition-name condition** -- tests whether a condition name (level 88) is true.
5. **Switch-status condition** -- tests the on/off status of an external switch (SPECIAL-NAMES).

Simple conditions can be combined with logical operators (AND, OR) and negated with NOT to form compound (combined) conditions. Parentheses may be used to override default precedence.

### Relation Conditions

A relation condition compares two operands using a relational operator. The general form is:

```
operand-1  IS [NOT]  relational-operator  operand-2
```

The relational operators are:

| Operator Phrase                 | Symbol | Meaning                  |
|---------------------------------|--------|--------------------------|
| IS EQUAL TO                     | =      | Equal                    |
| IS NOT EQUAL TO                 | NOT =  | Not equal                |
| IS GREATER THAN                 | >      | Greater than             |
| IS NOT GREATER THAN             | NOT >  | Not greater than (<=)    |
| IS LESS THAN                    | <      | Less than                |
| IS NOT LESS THAN                | NOT <  | Not less than (>=)       |
| IS GREATER THAN OR EQUAL TO     | >=     | Greater than or equal    |
| IS LESS THAN OR EQUAL TO        | <=     | Less than or equal       |

Operands can be identifiers, literals, arithmetic expressions, or index names. Comparison rules depend on the types of the operands:

- **Both numeric**: algebraic comparison regardless of USAGE or length.
- **Both alphanumeric/alphabetic**: character-by-character comparison from left to right, shorter operand padded with spaces on the right.
- **One numeric, one alphanumeric**: the numeric item is treated as an alphanumeric string for comparison if it is an integer display item; otherwise the comparison is not valid without an explicit conversion.
- **Group items**: always treated as alphanumeric regardless of subordinate item types.

### Class Conditions

A class condition tests the content of a data item. The general form is:

```
identifier-1 IS [NOT] class-name
```

Built-in class tests:

| Class Test     | True When                                              |
|----------------|--------------------------------------------------------|
| NUMERIC        | Contains only digits 0-9, with optional sign           |
| ALPHABETIC     | Contains only letters A-Z, a-z, and spaces             |
| ALPHABETIC-UPPER | Contains only uppercase letters A-Z and spaces       |
| ALPHABETIC-LOWER | Contains only lowercase letters a-z and spaces       |
| DBCS           | Contains only valid DBCS characters (mainframe)        |
| KANJI          | Contains only valid Kanji characters (mainframe)       |

You can also define custom classes in the SPECIAL-NAMES paragraph of the ENVIRONMENT DIVISION:

```cobol
ENVIRONMENT DIVISION.
CONFIGURATION SECTION.
SPECIAL-NAMES.
    CLASS VALID-DIGIT IS "0" THRU "9" "A" THRU "F".
```

Then use:

```cobol
IF WS-HEX-CHAR IS VALID-DIGIT
    PERFORM PROCESS-HEX
END-IF
```

Important rules for class conditions:
- NUMERIC can be tested on alphanumeric, numeric display, or packed-decimal items.
- An unsigned numeric display item is NUMERIC if it contains only digits.
- A signed numeric display item is NUMERIC if it contains digits with a valid operational sign.
- ALPHABETIC, ALPHABETIC-UPPER, and ALPHABETIC-LOWER cannot be tested on numeric items.
- Group items are treated as alphanumeric for class-condition testing.

### Sign Conditions

A sign condition tests whether the value of a numeric operand is positive, negative, or zero:

```
operand IS [NOT] POSITIVE / NEGATIVE / ZERO
```

The rules are:
- **POSITIVE** is true when the value is greater than zero.
- **NEGATIVE** is true when the value is less than zero.
- **ZERO** is true when the value is equal to zero.

The operand must be a numeric identifier or an arithmetic expression.

### Condition Names (88 Levels)

A condition name is a user-defined name associated with a specific value or set of values of a data item. Condition names are declared at level 88 under the data item they describe. They act as boolean tests against the parent item's current value.

```cobol
01  WS-ACCOUNT-STATUS      PIC X(01).
    88  ACCT-ACTIVE         VALUE "A".
    88  ACCT-CLOSED         VALUE "C".
    88  ACCT-SUSPENDED      VALUE "S".
    88  ACCT-INACTIVE       VALUE "I" "D".
    88  ACCT-VALID          VALUE "A" "C" "S" "I" "D".
```

Key characteristics:
- Level 88 items do not occupy storage. They are purely conditional tests on their parent item.
- A single 88-level can specify multiple values (e.g., `VALUE "I" "D"`) or a range of values (e.g., `VALUE "A" THRU "Z"`).
- Both single values and THRU ranges can be combined in a single 88-level.
- When referenced in a condition, the 88-level name evaluates to true if the parent item currently holds any of the specified values.
- The SET statement can be used to set a condition name to TRUE, which moves the first VALUE to the parent item.

### Combined Conditions (AND / OR / NOT)

Simple conditions can be joined using logical operators to form combined (compound) conditions:

```
condition-1 AND condition-2
condition-1 OR condition-2
NOT condition-1
```

**Evaluation precedence** (highest to lowest):
1. Parenthesized expressions -- `( )`
2. NOT
3. AND
4. OR

Examples:

```cobol
IF WS-AGE > 18 AND WS-STATUS = "A"
    PERFORM PROCESS-ADULT
END-IF

IF WS-CODE = "X" OR WS-CODE = "Y" OR WS-CODE = "Z"
    PERFORM PROCESS-SPECIAL
END-IF

IF NOT (WS-AMOUNT = ZERO)
    PERFORM PROCESS-AMOUNT
END-IF
```

Without parentheses, AND binds more tightly than OR:

```cobol
*> This:
IF A = 1 OR B = 2 AND C = 3
*> Is equivalent to:
IF A = 1 OR (B = 2 AND C = 3)
```

### Abbreviated Combined Relation Conditions

COBOL allows abbreviating consecutive relation conditions that share the same subject (left operand) and/or relational operator. The compiler expands the abbreviated form by carrying forward the subject and operator from the preceding relation condition.

```cobol
*> Full form:
IF WS-CODE = "A" OR WS-CODE = "B" OR WS-CODE = "C"

*> Abbreviated -- subject and operator carried forward:
IF WS-CODE = "A" OR "B" OR "C"
```

More complex abbreviation with NOT:

```cobol
*> Full form:
IF WS-X > 10 AND WS-X < 100

*> Abbreviated -- subject carried forward:
IF WS-X > 10 AND < 100
```

The abbreviated form works by inheriting the subject (and optionally the operator) from the leftmost complete relation condition in the sequence. This applies only within a single combined relation condition connected by AND or OR.

### Negated Conditions

Any condition (simple or combined) can be negated:

```cobol
IF NOT WS-EOF
    PERFORM READ-NEXT-RECORD
END-IF

IF NOT (WS-STATUS = "A" OR WS-STATUS = "C")
    PERFORM HANDLE-EXCEPTION
END-IF
```

When NOT is applied to a combined condition in parentheses, De Morgan's laws apply logically:
- `NOT (A AND B)` is true when A is false or B is false.
- `NOT (A OR B)` is true when A is false and B is false.

Note: NOT placed directly before a relational operator (e.g., `IS NOT EQUAL TO`) is part of the relation condition itself, not a logical negation of the entire condition.

## Syntax & Examples

### IF / ELSE / END-IF

The IF statement is the primary conditional branching statement in COBOL.

**Syntax:**

```cobol
IF condition-1
    statement-1 ...
[ELSE
    statement-2 ...]
[END-IF]
```

**Basic examples:**

```cobol
*> Simple IF
IF WS-BALANCE > 0
    DISPLAY "Account has positive balance"
END-IF

*> IF with ELSE
IF WS-BALANCE > 0
    DISPLAY "Positive balance: " WS-BALANCE
ELSE
    DISPLAY "Zero or negative balance"
END-IF
```

**CONTINUE and NEXT SENTENCE:**

CONTINUE is a no-operation statement useful as a placeholder:

```cobol
IF WS-AMOUNT = ZERO
    CONTINUE
ELSE
    PERFORM PROCESS-AMOUNT
END-IF
```

NEXT SENTENCE transfers control to the first statement after the next period (sentence terminator). It predates END-IF and should be avoided in modern code because it ignores the IF/END-IF structure entirely, jumping past the next period regardless of nesting.

```cobol
*> Legacy style -- avoid in new code
IF WS-CODE = SPACES
    NEXT SENTENCE
ELSE
    PERFORM PROCESS-CODE.
```

### Nested IF Statements

IF statements can be nested within either the IF branch or the ELSE branch:

```cobol
IF WS-TRANS-TYPE = "C"
    IF WS-AMOUNT > WS-CREDIT-LIMIT
        MOVE "OVER LIMIT" TO WS-MESSAGE
    ELSE
        PERFORM APPLY-CREDIT
    END-IF
ELSE
    IF WS-TRANS-TYPE = "D"
        PERFORM APPLY-DEBIT
    ELSE
        MOVE "INVALID TYPE" TO WS-MESSAGE
    END-IF
END-IF
```

Each END-IF explicitly closes its corresponding IF, eliminating the classic "dangling else" ambiguity. In period-delimited (pre-structured) COBOL, the ELSE is associated with the nearest preceding unpaired IF.

Deeply nested IFs (more than three levels) should generally be replaced with EVALUATE or refactored into separate PERFORMed paragraphs for readability.

### EVALUATE / WHEN / END-EVALUATE

EVALUATE is COBOL's multi-branch conditional, analogous to switch/case in other languages but significantly more powerful. It was introduced with COBOL-85.

**Syntax:**

```cobol
EVALUATE subject-1 [ALSO subject-2] ...
    WHEN phrase-1 [ALSO phrase-2] ...
        statement-1 ...
    [WHEN phrase-3 [ALSO phrase-4] ...
        statement-2 ...]
    [WHEN OTHER
        statement-3 ...]
END-EVALUATE
```

The subject can be:
- An identifier (data item)
- A literal
- An expression
- TRUE or FALSE

Each WHEN phrase specifies one or more values, ranges, or conditions to match against the subject.

**Basic example:**

```cobol
EVALUATE WS-REGION-CODE
    WHEN "N"
        PERFORM PROCESS-NORTH
    WHEN "S"
        PERFORM PROCESS-SOUTH
    WHEN "E"
        PERFORM PROCESS-EAST
    WHEN "W"
        PERFORM PROCESS-WEST
    WHEN OTHER
        PERFORM PROCESS-UNKNOWN-REGION
END-EVALUATE
```

Execution: the WHEN phrases are evaluated in order from top to bottom. The first matching WHEN executes its associated statements, then control passes to the statement after END-EVALUATE. There is no fall-through between WHEN branches (unlike C's switch without break).

Multiple values in a single WHEN:

```cobol
EVALUATE WS-DAY-CODE
    WHEN 1
    WHEN 7
        MOVE "WEEKEND" TO WS-DAY-TYPE
    WHEN 2 THRU 6
        MOVE "WEEKDAY" TO WS-DAY-TYPE
    WHEN OTHER
        MOVE "INVALID" TO WS-DAY-TYPE
END-EVALUATE
```

Note that stacking multiple WHEN phrases (WHEN 1 / WHEN 7) without intervening statements acts like an OR -- if either matches, the next block of statements executes.

### EVALUATE TRUE

When the subject is TRUE, each WHEN phrase contains a condition expression. This is the idiomatic COBOL way to express multi-way branching on arbitrary conditions:

```cobol
EVALUATE TRUE
    WHEN WS-AGE < 13
        MOVE "CHILD" TO WS-CATEGORY
    WHEN WS-AGE < 18
        MOVE "TEENAGER" TO WS-CATEGORY
    WHEN WS-AGE < 65
        MOVE "ADULT" TO WS-CATEGORY
    WHEN WS-AGE >= 65
        MOVE "SENIOR" TO WS-CATEGORY
    WHEN OTHER
        MOVE "UNKNOWN" TO WS-CATEGORY
END-EVALUATE
```

Similarly, EVALUATE FALSE tests for the first WHEN whose condition is false:

```cobol
EVALUATE FALSE
    WHEN WS-FILE-STATUS = "00"
        PERFORM HANDLE-FILE-ERROR
END-EVALUATE
```

### EVALUATE with ALSO

ALSO allows testing multiple subjects simultaneously. Each subject has a corresponding value in each WHEN phrase:

```cobol
EVALUATE WS-GENDER ALSO WS-AGE-GROUP
    WHEN "M" ALSO "ADULT"
        PERFORM PROCESS-ADULT-MALE
    WHEN "M" ALSO "CHILD"
        PERFORM PROCESS-CHILD-MALE
    WHEN "F" ALSO "ADULT"
        PERFORM PROCESS-ADULT-FEMALE
    WHEN "F" ALSO "CHILD"
        PERFORM PROCESS-CHILD-FEMALE
    WHEN OTHER
        PERFORM PROCESS-DEFAULT
END-EVALUATE
```

The keyword ANY in a WHEN phrase matches any value for that subject position:

```cobol
EVALUATE WS-TRANS-TYPE ALSO TRUE
    WHEN "C" ALSO WS-AMOUNT > 0
        PERFORM CREDIT-POSITIVE
    WHEN "C" ALSO WS-AMOUNT <= 0
        PERFORM CREDIT-ZERO-NEG
    WHEN "D" ALSO ANY
        PERFORM DEBIT-PROCESS
    WHEN OTHER
        PERFORM UNKNOWN-TRANS
END-EVALUATE
```

### EVALUATE with THRU

THRU (or THROUGH) specifies a range of values:

```cobol
EVALUATE WS-SCORE
    WHEN 90 THRU 100
        MOVE "A" TO WS-GRADE
    WHEN 80 THRU 89
        MOVE "B" TO WS-GRADE
    WHEN 70 THRU 79
        MOVE "C" TO WS-GRADE
    WHEN 60 THRU 69
        MOVE "D" TO WS-GRADE
    WHEN 0 THRU 59
        MOVE "F" TO WS-GRADE
    WHEN OTHER
        MOVE "?" TO WS-GRADE
END-EVALUATE
```

The first value in a THRU range must be less than or equal to the second value. Both endpoints are inclusive.

### Condition Names in Practice

Condition names (88 levels) are one of the most powerful and idiomatic features of COBOL conditional logic. They assign meaningful names to data values, making conditions self-documenting.

**Defining condition names:**

```cobol
01  WS-FILE-STATUS         PIC XX.
    88  FILE-OK            VALUE "00".
    88  FILE-EOF           VALUE "10".
    88  FILE-NOT-FOUND     VALUE "35".
    88  FILE-ALREADY-OPEN  VALUE "41".
    88  FILE-READ-ERROR    VALUE "46" "47".

01  WS-NUMERIC-RANGE       PIC 9(03).
    88  LOW-RANGE          VALUE 0 THRU 100.
    88  MID-RANGE          VALUE 101 THRU 500.
    88  HIGH-RANGE         VALUE 501 THRU 999.

01  WS-YES-NO-FLAG         PIC X(01).
    88  YES-FLAG           VALUE "Y" "y".
    88  NO-FLAG            VALUE "N" "n".
```

**Using condition names in IF:**

```cobol
READ INPUT-FILE INTO WS-INPUT-RECORD
    AT END SET FILE-EOF TO TRUE
END-READ

IF FILE-EOF
    PERFORM CLOSE-FILES
END-IF

IF NOT FILE-OK
    DISPLAY "File error: " WS-FILE-STATUS
    PERFORM ABORT-PROGRAM
END-IF
```

**Using condition names in EVALUATE:**

```cobol
EVALUATE TRUE
    WHEN FILE-OK
        PERFORM PROCESS-RECORD
    WHEN FILE-EOF
        PERFORM END-OF-FILE
    WHEN FILE-NOT-FOUND
        DISPLAY "File not found"
        PERFORM ABORT-PROGRAM
    WHEN OTHER
        DISPLAY "Unexpected status: " WS-FILE-STATUS
        PERFORM ABORT-PROGRAM
END-EVALUATE
```

### SET for Condition Names

The SET statement assigns the first VALUE of a condition name to its parent data item:

```cobol
SET FILE-EOF TO TRUE
*> Equivalent to: MOVE "10" TO WS-FILE-STATUS

SET YES-FLAG TO TRUE
*> Equivalent to: MOVE "Y" TO WS-YES-NO-FLAG
```

SET ... TO TRUE always moves the first value listed in the 88-level VALUE clause. If the 88 has multiple values (e.g., VALUE "Y" "y"), SET TO TRUE uses the first one ("Y").

SET ... TO FALSE is supported in some compilers when a FALSE IS clause is coded on the 88-level:

```cobol
01  WS-FOUND-FLAG          PIC X(01).
    88  RECORD-FOUND       VALUE "Y"
                           FALSE IS "N".
```

```cobol
SET RECORD-FOUND TO TRUE     *> Moves "Y"
SET RECORD-FOUND TO FALSE    *> Moves "N"
```

### Complex Compound Conditions

Real-world COBOL programs often combine multiple condition types:

```cobol
IF (ACCT-ACTIVE OR ACCT-SUSPENDED)
   AND WS-BALANCE > 0
   AND WS-LAST-ACTIVITY NOT = SPACES
    PERFORM GENERATE-STATEMENT
END-IF

IF WS-INPUT IS NUMERIC
   AND WS-INPUT NOT = ZERO
   AND (WS-CATEGORY = "A" OR "B" OR "C")
    PERFORM VALID-INPUT-PROCESSING
END-IF
```

Using parentheses for clarity in compound conditions is strongly recommended even when operator precedence would produce the correct result, because it makes the intent explicit to maintenance programmers.

## Common Patterns

### Flag-driven processing loops

```cobol
01  WS-EOF-FLAG            PIC X(01) VALUE "N".
    88  END-OF-FILE        VALUE "Y".
    88  NOT-END-OF-FILE    VALUE "N".

PERFORM UNTIL END-OF-FILE
    READ INPUT-FILE INTO WS-RECORD
        AT END
            SET END-OF-FILE TO TRUE
        NOT AT END
            PERFORM PROCESS-RECORD
    END-READ
END-PERFORM
```

### Validation with 88 levels

```cobol
01  WS-STATE-CODE          PIC XX.
    88  VALID-STATE        VALUE "AL" "AK" "AZ" "AR" "CA"
                                 "CO" "CT" "DE" "FL" "GA"
                                 *> ... remaining states ...
                                 "WV" "WI" "WY".

IF NOT VALID-STATE
    MOVE "INVALID STATE CODE" TO WS-ERROR-MSG
    PERFORM WRITE-ERROR
END-IF
```

### EVALUATE TRUE for multi-condition branching (replacing nested IFs)

```cobol
EVALUATE TRUE
    WHEN WS-AMOUNT > 10000
     AND ACCT-ACTIVE
        PERFORM LARGE-ACTIVE-PROCESS
    WHEN WS-AMOUNT > 10000
     AND ACCT-SUSPENDED
        PERFORM LARGE-SUSPENDED-PROCESS
    WHEN WS-AMOUNT > 0
        PERFORM STANDARD-PROCESS
    WHEN WS-AMOUNT = 0
        PERFORM ZERO-AMOUNT-PROCESS
    WHEN WS-AMOUNT < 0
        PERFORM NEGATIVE-AMOUNT-PROCESS
END-EVALUATE
```

### EVALUATE as a state machine dispatcher

```cobol
EVALUATE WS-PROGRAM-STATE
    WHEN "INIT"
        PERFORM INITIALIZATION
        MOVE "READ" TO WS-PROGRAM-STATE
    WHEN "READ"
        PERFORM READ-INPUT
        IF FILE-EOF
            MOVE "DONE" TO WS-PROGRAM-STATE
        ELSE
            MOVE "PROC" TO WS-PROGRAM-STATE
        END-IF
    WHEN "PROC"
        PERFORM PROCESS-RECORD
        MOVE "WRITE" TO WS-PROGRAM-STATE
    WHEN "WRITE"
        PERFORM WRITE-OUTPUT
        MOVE "READ" TO WS-PROGRAM-STATE
    WHEN "DONE"
        PERFORM WRAP-UP
    WHEN OTHER
        DISPLAY "Invalid state: " WS-PROGRAM-STATE
        PERFORM ABORT-PROGRAM
END-EVALUATE
```

### Guarding numeric operations with class conditions

```cobol
IF WS-INPUT-AMOUNT IS NUMERIC
    COMPUTE WS-TAX = WS-INPUT-AMOUNT * WS-TAX-RATE
ELSE
    MOVE "NON-NUMERIC AMOUNT" TO WS-ERROR-MSG
    PERFORM WRITE-ERROR
END-IF
```

### Combining condition names with relation conditions

```cobol
IF ACCT-ACTIVE
   AND WS-DAYS-SINCE-ACTIVITY < 365
   AND WS-BALANCE >= WS-MINIMUM-BALANCE
    MOVE "GOOD STANDING" TO WS-ACCT-MESSAGE
ELSE
    IF ACCT-ACTIVE
       AND WS-DAYS-SINCE-ACTIVITY >= 365
        MOVE "DORMANT" TO WS-ACCT-MESSAGE
    ELSE
        MOVE "REVIEW NEEDED" TO WS-ACCT-MESSAGE
    END-IF
END-IF
```

### Defensive WHEN OTHER

Always include WHEN OTHER in EVALUATE statements as a safety net for unexpected values, even if you believe all cases are covered. This catches data corruption, new values added upstream, or migration issues.

## Gotchas

- **Period terminates ALL nested IFs.** In older period-delimited code (before END-IF was available), a single period closes every open IF at that point. If you add a period inside a nested IF by mistake, it terminates the entire IF/ELSE structure prematurely, causing logic errors that are extremely difficult to spot. Always use explicit END-IF scope terminators.

- **NEXT SENTENCE vs CONTINUE.** NEXT SENTENCE jumps to the statement after the next period, completely ignoring IF/END-IF nesting boundaries. CONTINUE does nothing and allows flow to proceed into the ELSE branch or past END-IF as expected. Using NEXT SENTENCE inside an END-IF delimited structure almost always produces unintended behavior. Use CONTINUE instead.

- **Abbreviated conditions can mislead.** The expression `IF A = 1 OR 2` is valid COBOL meaning `IF A = 1 OR A = 2`. However, `IF A = 1 OR B` does NOT mean `IF A = 1 OR B IS TRUE` -- it means `IF A = 1 OR A = B`. This is because the abbreviated combined relation condition inherits the subject (A) and the operator (=) from the left. When B is a data item, it becomes the object of the inherited comparison. This is a frequent source of bugs.

- **NOT with abbreviated conditions.** `IF A NOT = 1 OR 2` expands to `IF A NOT = 1 OR A NOT = 2`, which is always true (every value of A is either not-1 or not-2). The programmer almost certainly intended `IF NOT (A = 1 OR 2)` which expands to `IF NOT (A = 1 OR A = 2)`, meaning A is neither 1 nor 2. Always use parentheses with NOT on abbreviated conditions.

- **AND/OR precedence without parentheses.** `IF A = 1 OR B = 2 AND C = 3` evaluates as `IF A = 1 OR (B = 2 AND C = 3)` because AND has higher precedence than OR. Many programmers read it left-to-right and assume the OR applies before the AND. Always use parentheses in compound conditions to make intent explicit.

- **EVALUATE falls through stacked WHENs.** Multiple consecutive WHEN clauses without intervening statements act as a logical OR. This is intentional, but be careful not to accidentally stack WHENs by leaving a blank line between a WHEN and its statements -- the compiler does not care about blank lines, so the statements still belong to the nearest preceding WHEN.

- **Class condition on signed numeric fields.** A signed numeric display field containing a trailing overpunch sign may fail an IS NUMERIC test on some compilers if the sign configuration does not match. Ensure SIGN IS LEADING/TRAILING SEPARATE CHARACTER is understood when testing signed fields with class conditions.

- **Comparing numeric and alphanumeric items.** Comparing a numeric item to an alphanumeric item invokes non-numeric comparison rules, which compare character by character. The value 9 in PIC 9(3) is stored as "009" and compares differently than you might expect against an alphanumeric "9  ". Always ensure operand types match or explicitly convert before comparing.

- **EVALUATE TRUE with overlapping conditions.** Only the first matching WHEN executes, so order matters. Placing a broad condition before a narrow one shadows the narrow condition. For example, placing `WHEN WS-AGE > 0` before `WHEN WS-AGE > 65` means the senior condition never triggers.

- **SET TO TRUE with multi-value 88 levels.** When an 88-level has multiple values (e.g., VALUE "Y" "y"), SET TO TRUE always moves the first value. If your program later tests with `IF WS-FLAG = "y"`, that test fails. Be aware which value SET TO TRUE assigns.

- **Spaces and LOW-VALUES in conditions.** `IF WS-FIELD = SPACES` is not the same as `IF WS-FIELD = LOW-VALUES`. An uninitialized field in WORKING-STORAGE may contain LOW-VALUES (binary zeros) which is distinct from spaces (X'40' in EBCDIC, X'20' in ASCII). Always initialize fields or test for both if the initial state is uncertain.

- **Condition names on REDEFINES.** If a data item is redefined, 88-level conditions on the original item test the raw bytes, which may not correspond to the redefined interpretation. This can cause subtle logic errors when the same storage is accessed through different definitions.

## Related Topics

- **data_types.md** -- Understanding COBOL data types (PIC clauses, USAGE, numeric vs. alphanumeric) is essential for knowing which comparison rules apply in relation conditions and which class conditions are valid.
- **paragraph_flow.md** -- IF and EVALUATE control which paragraphs are PERFORMed; understanding paragraph and section flow is necessary to trace conditional execution paths.
- **working_storage.md** -- Level 88 condition names are declared in WORKING-STORAGE (or LINKAGE SECTION) under their parent data items; the structure and initialization of working storage directly affects conditional logic behavior.
- **table_handling.md** -- Conditional logic is frequently used with table operations: subscript bounds checking, SEARCH/SEARCH ALL rely on conditions, and 88-levels can be defined on table elements for value validation.
