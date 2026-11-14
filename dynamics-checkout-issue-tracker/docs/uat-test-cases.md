# Checkout Issue Tracker: UAT test cases

Version tested: 1.1.0.1 (first run); retests on the build released as 1.1.0.2
Environment: trial-eu-60c28b68
Tester: Yuka Gega
Test date: 14 / 11 / 2026

Result key: Pass / Fail / Pass with note / Retest

## User stories under test

- US1 (Staff): log a failed scan quickly, so the queue keeps moving.
- US2 (Supervisor): be alerted when one machine has repeated failures, so I can escalate early.
- US3 (Manager): see failures by machine and over time, so I can show the vendor the impact.

## Test cases

| ID | Story | Role | Steps | Expected result | Actual result | Result | Evidence |
| --- | --- | --- | --- | --- | --- | --- | --- |
| TC01 | US1 | Staff | New issue. Fill Issue title only. Save. | Save blocked; Machine and Error type flagged as required. | Save blocked. "Machine: Required fields must be filled in" and "Error type: Required fields must be filled in". | Pass | TC01.png |
| TC02 | US1 | Staff | New issue with title, machine and error type. Save. | Saves; Issue status is New automatically; Owner is the staff user. | First run: saved without an Issue date (DEF-01). Retest after Rule 4: saving without a date blocked with "Enter the date and time of the issue."; Status New and Owner Staff Test as expected. | Pass (retest) | TC02.png (retest) |
| TC03 | US1 | Supervisor | Open an issue. Set status to Resolved, leave Resolution note empty. Save. | Save blocked with "Add a resolution note before resolving this issue." | Save blocked, but with the standard message "Resolution note: Required fields must be filled in." (Rule 1's required check fired first; Rule 2's custom message did not appear on the form). | Pass with note | TC03.png |
| TC04 | US1 | Supervisor | Same issue: add a resolution note. Save. | Saves as Resolved. | Saved as Resolved with a note. Retested after the Rule 1 change was reverted: "Card was damaged" saved as Resolved with note "Manual entry". | Pass (retest) | TC04.png, TC04-retest.png |
| TC05 | US1 | Admin | In the maker data grid (outside the app), set an issue to Resolved with no note. | Row not saved; rule enforced outside the form. | "1 row is not saved" for an imported issue set to Resolved with an empty note. | Pass | TC05.png |
| TC06 | US1 | Staff | Sign in as Staff Test. Open Checkout Issues. | Only issues owned by Staff Test are visible. | 2 issues shown, both owned by Staff Test (57 exist in total). | Pass | TC06.png |
| TC07 | US1 | Staff | Select an own issue, then open it. Look for Delete and Deactivate. | Neither Delete nor Deactivate appears in the list or on the record. | List with a row selected: no Delete or Deactivate in the command bar or the ⋮ menu. Record view not captured in this run (passed on Day 6). | Pass with note | TC07.png |
| TC08 | US1 | Staff | Open any machine. | "Read-only: You don't have access to edit this record." | SCO-1 opened with the read-only banner; no Save or Delete. | Pass | TC08.png |
| TC09 | US2 | Staff/Admin | Find a machine with 4 open issues. Add a 5th open issue on it. | Flow runs; the 5th issue becomes Escalated; earlier issues unchanged. | "Card was damaged" on SCO-2 created as New, then changed to Escalated by the flow. | Pass | TC09-1.png, TC09-2.png |
| TC10 | US2 | Staff/Admin | Add a 6th open issue on the same machine. | Flow runs; condition false; the 6th issue stays New. | "Failed reading of a card" on SCO-2 stayed New. | Pass | TC10.png |
| TC11 | US1 | Admin | Import a CSV row with an unknown machine (SCO-5) and a row with no Error type. | Both rows fail with clear reasons in the import Failures tab. | First run: two rows failed at the mapping stage because Error type was left unmapped (OBS-02). Retest with 4 rows and Error type mapped manually (Card broken set to Ignore): all 4 failed for their own reason. SCO-5: lookup not resolved; Resolved without note: "cannot be NULL" (Rule 1, server-side); invalid Error type: "Select an error type." (Rule 3); no date: "Enter the date and time of the issue." (Rule 4). | Pass (retest) | TC11.png, TC11-retest.png |
| TC12 | US3 | Supervisor | Open Supervisor dashboard. | Charts show open issues by machine, by status and per day; Escalated list matches the Escalated view. | Open by machine: SCO-1 7, SCO-2 6, SCO-3 5, SCO-4 5. Status: Escalated 4, New 18, Resolved 35 (57 total). Escalated list shows the same 4 issues. Per-day chart includes a "(blank)" bar (see DEF-01). | Pass with note | TC12.png |
| TC13 | US3 | Staff | Open Supervisor dashboard as Staff Test. | Charts and list show only Staff Test's own issues. | SCO-1 1, SCO-2 1; New 2; Escalated list empty. Per-day chart shows "(blank)" for the undated issue. | Pass | TC13.png |
| TC14 | US3 | Admin | Change an issue's status, then open Related > Audit History. | Entry shows who, when, old value and new value. | First screenshot showed only the Create event. Retest: Update row recorded, Issue status New to Escalated, by Yuka Gega at 20:54. | Pass (retest) | TC14.png (retest) |

## Summary

- Pass: 12 / 14 (including 4 passed on retest: TC02, TC04, TC11, TC14)
- Pass with note: 2 / 14 (TC03, TC07)
- Fail: 0
- First run on 1.1.0.1; fixes and retests released as 1.1.0.2

## Defects and observations

- DEF-01 (Low), FIXED: Issue date was not required, so issues could be saved without a date. Found in TC02; showed as "(blank)" in the per-day chart (TC12, TC13). Fix: new business rule "Require issue date" (Entity scope). Retested in TC02 and TC11; the undated issue was given a date.
- DEF-02 (Medium), FIXED before release: while trying to change the TC03 message, a Show Error Message action was added to Rule 1's Resolved branch. It would have blocked every resolve, even with a note. Reverted, and TC04 retested.
- OBS-01: When an issue is resolved without a note on the form, the standard required-field message appears instead of the custom Rule 2 message (TC03). Behaviour is correct; only the wording differs.
- OBS-02: If any value in a choice column can't be mapped, the import wizard leaves the whole column unmapped, so valid values in other rows also fail (TC11). Admins must map choice values manually and set invalid ones to Ignore.

## Release

- Version 1.1.0.2 exported as managed and unmanaged packages on 14/11/2026, after all retests passed.
- Solution checker run on export: 0 issues (Critical, High, Medium, Low).
- Managed dependencies unchanged from 1.1.0.1: App Framework Infra Extensions, System App Actions, App Module Assets.
- The package also contains Dataverse search index components (DVTableSearch) added automatically by the platform; they add no new dependencies, so they were kept.

## Known limitations (accepted, not defects)

- Hiding Deactivate is a UI-level control; Write permission still exists.
- Days with zero issues don't appear in the per-day chart.
- Staff can open the Supervisor dashboard (data is filtered by security).
