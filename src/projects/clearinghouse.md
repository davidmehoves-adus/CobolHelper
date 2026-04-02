# Clearinghouse

## Overview

A monthly batch reporting process used by the 32 public colleges and universities in Minnesota. It generates a student enrollment report that is transmitted to the National Student Clearinghouse (NSC), which serves as a service provider to the National Student Loan Data System (NSLDS). The process supports compliance with federal reporting requirements and facilitates degree and enrollment verification for students.

## Environment

- **Operating System**: Red Hat Enterprise Linux 9
- **Database**: Oracle SQL
- **COBOL Version**: Micro Focus Visual COBOL (COBOL 9 engine)
- **Scheduler**: JAMS — runs monthly, typically on the 15th

## Key Programs

| Program | Role |
|---|---|
| FA0300CB | Scheduled batch process — entry point. Calls FA0155CB for certain input. |
| FA0155CB | Called by FA0300CB for specific input processing. |

## Error Handling

Abend-type errors are logged to a database table.

## History

The code was originally created a long time ago but is actively maintained. Updates occur as reporting requirements change.

## Open Questions

- Is this running on a mainframe? (check Janke docs)
- What is the full call chain beyond FA0300CB → FA0155CB?
- What is the structure of the enrollment report output?
- What database tables are involved?
