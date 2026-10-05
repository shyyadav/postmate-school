---
name: school-api
description: Application knowledge, test data rules and regression scenarios for the School API (fees). Use when running fee regression, smoke checks or triage against the School API with Postmate saved requests.
---

# School API: QA Skill

This skill tells you how the School API behaves and how to test it.
You test by sending saved requests from the Postmate "School API" collection
with the Postmate MCP `send_request` tool, and judging the responses against
the rules and scenarios below. There are no test scripts.

## Application knowledge

### General
- Requests are addressed as "School API.<Request Name>", e.g. "School API.Add Fee".
- Authentication is handled by request chaining (Login runs as a pre-request).
  Never send Login yourself unless a step fails with 401, and never print the token.
- The base URL and credentials live in the Postmate environment. Never hard-code them.

### Fees
- A fee has a name, a type, an amount, and one or more grades it applies to.
- Valid fee types: "Annually", "Monthly", "One Time" (with a space).
- Grades are named in words (e.g. "Eighth", "Tenth"), not numbers.
- Fee names are unique within a school.
- On save, the API appends the amount to the fee name, by design.
  Example: "QA Lab Fee" with amount 750 is stored as "QA Lab Fee 750".
  Use the stored name ("{submitted name} {amount}") to find or delete the fee.
- "Add Fee" sends grades in `grade` (an array). Fee responses return them in
  `grades`. Compare grades as a set: order does not matter.
- A fee applies only to the grades it was created for. It must NOT appear
  when fetching fees for any other grade.
- Deleting a fee removes it everywhere: from the full fee list and from
  every grade.
- Do not assert on `monthlyAmount` or `remaingDays` unless a scenario says so.

## Saved requests used

| Request | Purpose | Values to pass as data |
|---|---|---|
| School API.Add Fee | Create a fee | `feeName`, `feeType`, `amount`, `grade` |
| School API.Get All Fees | List every fee | none |
| School API.Get Fees By Grade | List fees for one grade | `grade` |
| School API.Delete Fee | Delete a fee by its stored name | `feeName` (the stored name) |

## Test data rules

- Test data lives in the Postmate data table `School-fee`:
  `_dtag,feeName,feeType,amount,grade,otherGrade`
- Run a scenario once for every row whose `_dtag` matches the requested tag
  (e.g. "fee-*" means every fee row).
- `otherGrade` is a grade the fee was NOT created for. Use it for the
  negative check.
- Pass row values as data overrides to `send_request`. Never hard-code values
  that exist in the data table.
- Make every created fee name unique per run: append a suffix with the date,
  time and row number to the row's `feeName`
  (e.g. "QA Lab Fee 20260926-0510-01"), so repeated runs never collide.

## Scenario 1: Fee lifecycle

Requests: "School API.Add Fee", "School API.Get All Fees",
"School API.Get Fees By Grade", "School API.Delete Fee".

1. Create the fee with "Add Fee" using the row's values (with the unique suffix).
   Expected: 200.
2. Fetch all fees with "Get All Fees".
   Expected: the new fee is present with name "{submitted name} {amount}",
   and the same type, amount and grades.
3. Fetch fees for the row's `grade` with "Get Fees By Grade".
   Expected: the new fee is present.
4. Fetch fees for the row's `otherGrade` with "Get Fees By Grade".
   Expected: the new fee is absent.
5. Delete the fee with "Delete Fee".
   Expected: 200.
6. Fetch all fees again.
   Expected: the fee is no longer present.
7. Fetch fees for the row's `grade` again.
   Expected: the fee is no longer present.

Cleanup: if any step fails after step 1, still run step 5 so the fee does
not stay in the system.

## Safety rules

- Only delete records created during this run. Never modify or delete records
  that existed before the run, even if they look like old test data.
- If you cannot safely identify the record to delete, stop and report it.
  Do not guess.
- If a saved request looks misconfigured (hard-coded URL, header or value
  instead of a variable), do not run it. Mark the affected steps BLOCKED and
  report the request and the problem.
- Do not guess. If a response is unclear, say so instead of marking PASS.

## Report format

Always produce this report, even when every step passes.

### Summary
- Rows run: N
- Requests sent: N
- Steps: N passed, N failed, N blocked
- Cleanup: every fee created in this run deleted and verified absent (yes/no)
- Duration: approximate

### Results

| Row (_dtag) | Step | Request | Expected | Actual | Result |
|---|---|---|---|---|---|
| fee-annual | 1 | Add Fee | 200 | 200 | PASS |

Result is one of PASS, FAIL, BLOCKED.

### Observations
Anything unexpected that did not fail a step (for example, inconsistent values
between endpoints, or leftover records from earlier runs). Do not mark these
as failures.