# ASCII Diagram Guide

Standard diagram formats for documentor output.

## Call Tree

```
FA0300CB
├── FA0155CB (certain input)
│   └── [returns to FA0300CB]
└── [report output]
```

## Process Flow

```
┌─────────────┐     ┌──────────────┐     ┌─────────────┐
│ Open Files   │────▶│ Process Loop │────▶│ Close Files  │
└─────────────┘     └──────┬───────┘     └─────────────┘
                           │
                    ┌──────▼───────┐
                    │ Control Break?│
                    ├── Yes ───────▶ Print Subtotals
                    └── No ────────▶ Accumulate
```

## Data Flow

```
[INPUT FILE] ──▶ READ ──▶ [WS-RECORD] ──▶ VALIDATE ──▶ WRITE ──▶ [OUTPUT FILE]
```

## Decision Tree

```
IS ACCOUNT-STATUS = 'A'?
├── YES ──▶ PROCESS-ACTIVE
│          ├── BALANCE > 0? ──▶ NORMAL-PROCESSING
│          └── BALANCE ≤ 0? ──▶ OVERDRAFT-CHECK
└── NO  ──▶ IS ACCOUNT-STATUS = 'C'?
            ├── YES ──▶ PROCESS-CLOSED
            └── NO  ──▶ PROCESS-ERROR
```

## Paragraph Flow (Sequence)

```
MAIN-PROCESS
│
├── 1. INITIALIZE-FIELDS
├── 2. OPEN-FILES
├── 3. READ-INPUT          ◄─── priming read
├── 4. PROCESS-LOOP ───────┐
│      ├── VALIDATE-RECORD │
│      ├── APPLY-RULES     │ PERFORM UNTIL EOF
│      ├── WRITE-OUTPUT    │
│      └── READ-INPUT ─────┘
├── 5. PRINT-TOTALS
└── 6. CLOSE-FILES
```
