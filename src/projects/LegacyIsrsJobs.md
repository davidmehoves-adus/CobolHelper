# Legacy ISRS Jobs

## Overview

ISRS (Integrated Statewide Record System) is an ERP and Student Information System used by the 32 public colleges and universities in Minnesota. All COBOL programs in this project run as ad-hoc or scheduled batch jobs — there is no interactive/online component. Jobs are managed through the JAMS scheduler or triggered manually as needed.

## Environment

- **Operating System**: Red Hat Enterprise Linux 9
- **Database**: Oracle SQL
- **COBOL Version**: Micro Focus Visual COBOL (COBOL 9 engine)
- **Scheduler**: JAMS

## Clearinghouse

A monthly batch reporting process that generates a student enrollment report transmitted to the National Student Clearinghouse (NSC), which serves as a service provider to the National Student Loan Data System (NSLDS). Supports compliance with federal reporting requirements and facilitates degree and enrollment verification for students.

Runs monthly, typically on the 15th.

### Key Programs

| Program | Role |
|---|---|
| FA0300CB | Scheduled batch process — entry point. Calls FA0155CB for certain input. |
| FA0155CB | Called by FA0300CB for specific input processing. |

### Error Handling

Abend-type errors are logged to a database table.

### History

The code was originally created a long time ago but is actively maintained. Updates occur as reporting requirements change.

### Open Questions

- What is the full call chain beyond FA0300CB → FA0155CB?
- What is the structure of the enrollment report output?
- What database tables are involved?

## Student Records

*To be populated as programs are identified.*

## Curriculum

*To be populated as programs are identified.*

## Financial Aid

Batch jobs related to federal financial aid processing — awards, grants, loans, and disbursements. These programs interact with the U.S. Department of Education's Common Origination and Disbursement (COD) system via TDClient/SAIG transmission, generating COD-compliant XML (CommonRecord 5.0c schema) for loan and grant reporting.

### Key Programs

| Program | Role |
|---|---|
| FA0017CP | Generates the Award vs Expenditure Roster — a 132-column report comparing awarded financial aid amounts against actual expenditures (EG transactions, payroll earnings, or loan checks) per student, award, and term. Supports Pell, ACG, SMART, work-study, loans, and general awards with distinct data retrieval paths for each type. |
| FA2641CF | Direct Loan origination. Extracts new Subsidized, Unsubsidized, PLUS, and Grad PLUS loans at status 20 (ready to send), builds PC-IMPORT records, generates COD XML, transmits to DOE, and advances loan status to 30 (exported). Uses a two-connection architecture (D1 read/staging, D2 write/status) for independent commit/rollback control. |
| FA2651CF | Pell Grant origination and disbursement reporting. Processes students from the ISIR pool, validates eligibility, creates origination records when data changes (transaction number, cost, EFC), and reports disbursements per term by comparing current payments against previously reported amounts. Supports early payment reporting and additional Pell (150% awards). |
| FA2655CF | Direct Loan disbursement download. Extracts disbursements at status 29 (aid applied), determines action type (D for new disbursement, A for adjustment with returns), builds PC-IMPORT records with enrollment status and CIP codes, generates COD XML, and transmits to DOE. Same two-connection architecture as FA2641CF. |
