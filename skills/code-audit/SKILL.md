---
name: code-audit
description: Use when asked to audit or review a pull request, a branch, a diff, a commit, or code from a junior developer or a student. Audits the change against the `architecture`, `security`, `performance` and `database-design` skills and returns a structured report: a verdict, then blockers, should-fix items and suggestions, each with the file and line, a concrete failure scenario and the fix. Has a teaching mode that explains the principle behind each finding.
---

> This skill is the process and the report format. The rules come from the `architecture`, `security`, `performance` and `database-design` skills; load the ones the change touches and don't restate their rules here.

## 1. Get the Whole Change
- A pull request: `gh pr diff <number>` and `gh pr view <number>` for the description. A branch: `git diff main...HEAD` (three dots, so you see only the branch's own changes). A commit: `git show <sha>`.
- Read the description or task first: review the change against what it was meant to do, not only against itself.
- Read every changed file in full, not just the diff hunks, plus the code the change calls and the code that calls it. Most real bugs sit just outside the hunk.

## 2. Review in Passes
Go through the change once per concern; mixing them hides findings.

1. **Correctness:** does it do what the task asked? Edge cases: empty input, zero, null, the first and last item, a second request at the same moment, a retry.
2. **Security:** load the `security` skill. Ownership checks, mass assignment, tenancy, uploads, webhooks, what the response exposes.
3. **Data:** load `architecture` and `database-design` when the change writes data or touches the schema. Transactions, side effects after commit, races, idempotency, status changes, types and constraints.
4. **Performance:** load `performance` when the change reads lists, relations or large tables.
5. **Structure:** thin controllers, the service layer, where the code lives.
6. **Tests:** do they cover the failure paths, not only the happy path? Would they fail if the bug they guard against came back?

Run the test suite when you can, and say so when you couldn't.

## 3. Only Report What You Can Defend
- Every finding names a file and line, says what goes wrong in a concrete scenario ("user B can open user A's invoice by changing the ID"), and shows the fix.
- Verify before reporting: read the surrounding code to confirm the problem is real and not handled elsewhere (a policy, a middleware, a global scope).
- Problems that existed before this change go in their own section, not mixed with what the change introduced.
- Don't report formatting or style that a linter (Pint, ESLint, Prettier) already enforces.
- If there's nothing to fix, say so plainly. Don't invent findings to fill the report.

## 4. Severity
| Level | Meaning | Examples |
|---|---|---|
| **Blocker** | Must be fixed before merge: wrong data, a security hole, or a crash | IDOR, money as float, email inside a transaction, a missing migration, a broken test |
| **Should fix** | Real cost if it ships, but not dangerous today | N+1 on a list that will grow, logic in a controller, a missing failure-path test |
| **Suggestion** | Better, optional | A clearer name, a smaller method, a simpler query |

A change with any blocker is never approved.

## 5. Report Format
```markdown
## Verdict: Request changes | Approve | Approve with suggestions
One or two sentences: what the change does and the main reason for the verdict.

## Blockers
1. **IDOR on invoice download** — `app/Http/Controllers/InvoiceController.php:42`
   Any signed-in user can download any invoice by changing the ID.
   Fix: load it through the owner: `$request->user()->invoices()->findOrFail($id)`.

## Should fix
...

## Suggestions
...

## Already there before this change
...

## Good
- One or two specific things done well, if any.

## Verified
- Tests: ran `php artisan test`, 214 passed. / Not run: no database available.
- Not checked: ...
```
Keep each finding short. Group repeats of the same problem into one finding with every location listed.

## 6. Teaching Mode
Use it when reviewing code from a junior developer, a student, or anyone learning, or when asked to teach.

- For each blocker and should-fix item, add one line of **why**: the principle behind the rule and what it protects against, in plain words.
- Point them to the fix; for a student, describe it and let them write it, unless they ask for the code.
- Limit suggestions to the two or three that teach the most. A list of twenty nitpicks teaches nothing.
- Name one thing they did well, and mean it.
- Write in the language the student uses.

## 7. Don't
- Don't edit the code during a review unless asked; the report is the output.
- Don't post comments on GitHub unless asked; show the report first.
- Don't approve because the tests pass. Tests only cover what someone thought of.
