# Dynamics Study Notes


## Daily log
Three lines a day: what I built, what broke, how I fixed it.

### Day 1 (8 Nov 2026): Setup
- Built: signed up for the Dynamics 365 Sales trial and confirmed the environment is Ready with Sales Hub and Customer Service Hub.
- Broke: the sign-up refused my Gmail address, then the Customer Service trial failed with an "invalid promotion code" error.
- Fixed: created a new work account through the "set up" link, then used the Sales trial instead, which worked. Also learned to switch away from the default environment before building.

### Day 2: Tables and columns
- Built: Created a solution with my own publisher (prefix yg), then the Checkout Issue table with User or team ownership, seven custom columns (including four Choice columns and a required Error type), and five test rows. Issue status defaults to New automatically.
- Broke: In the default environment the advanced table settings were greyed out because of limited privileges. Later, my test data showed inconsistent machine names ("SCO-1" vs "SCO 1"), which would break counts per machine.
- Fixed: Switched to my own trial environment, where I am System Administrator, and built everything inside the solution. The naming problem will be solved on Day 3 with a separate Machine table and a lookup, so staff pick a machine from a list instead of typing it.
- Note: Dataverse felt familiar from two years of Microsoft Access at DFEi. The new parts were ownership, built-in system columns and working inside solutions.

### Day 3: Relationships and forms
- Built: A Machine table with Organization ownership (machines are shared reference data, not owned by individual staff), four machines with location and software version, and a required Machine lookup on Checkout Issue with Restrict delete, so a machine with logged issues can't be deleted. Then a main form with two sections: Issue details and Resolution.
- Broke: Naming the primary column "Machine ID" failed, because it clashed with the system-generated primary key yg_machineid. I also nearly created a separate solution called "Machine" instead of a table, and adding a form section left me with extra empty sections.
- Fixed: Renamed the primary column to "Machine code", created the table inside my existing solution, and moved both resolution fields into one section before deleting the empty ones. My test rows now pick machines from a list, which fixes the "SCO-1" vs "SCO 1" problem from Day 2.
- Note: Every Dataverse table has a hidden GUID primary key, separate from the primary name column users see.

### Day 4: Model-driven app and views
- Built: Three views for Checkout Issue (Active, Open issues, Escalated issues) with chosen columns and newest-first sorting, an updated Machine view, and a model-driven app "Checkout Issue Tracker" with an Issue Tracking navigation group. Tested it in the app: saving without Machine or Error type is blocked, Issue status defaults to New, and each view filters correctly.
- Broke: Open issues hid the escalated record, because the view had two filters (not Resolved and not Escalated). Also, my Day 2 test dates had been saved as 11 September instead of 9 November, because the maker screen used US date format when I entered them, and times appeared an hour apart between screens.
- Fixed: Removed the extra filter, so Open issues means "anything not resolved". Corrected the test dates to 09/11/2026 and set the display format to English (Ireland). The time difference came from time zone settings, not wrong data.
- Note: Date format and time zone settings can silently corrupt data entry; check regional settings before entering or importing data.

### Day 5: Business rules
- Built: Two business rules on Checkout Issue, both with Entity scope. Rule 1 ("Require resolution note when resolved") makes Resolution note required when Issue status is Resolved and optional otherwise. Rule 2 ("Block resolve without note") shows my own error message, "Add a resolution note before resolving this issue", when an issue is Resolved and the note is empty.
- Tested: In the app, resolving without a note was blocked with both the red asterisk and my message, and saving worked once a note was added. In the maker data grid, outside any form, the same change failed with "1 row is not saved", so Rule 2 protects the data itself, not just the form.
- Broke: While building Rule 2, the second condition first used "Equals" with an empty value, which is unreliable for checking an empty field.
- Fixed: Changed the operator to "Does not contain data" and joined both conditions with AND.
- Note: "Set Business Required" mainly guides users on forms; "Show error message" with Entity scope enforces the rule wherever the data is changed.

### Day 6: Security
- Built: A test user (Staff Test) with a Dynamics 365 trial licence, and two security roles copied from Basic User. Checkout Staff: Create, Read, Write, Append on Checkout Issue at User level, no Delete; Read and Append To on Machine at Organization level. Checkout Supervisor: full Organization access to issues including Assign, and can manage machines; no Delete for either role. Assigned Checkout Staff to the test user and shared the app with both roles.
- Tested (signed in as Staff Test in a private window): The issue list was empty because all existing issues belonged to me; Staff Test could create an issue and pick a machine; there was no Delete button; machines opened as "Read-only: You don't have access to edit this record". From my admin account I could see Staff Test's issue with Staff Test as Owner, plus Delete and Assign.
- Gap found: Staff can still Deactivate their own issues, because deactivating only needs Write permission. A deactivated issue disappears from Active views, so it works like a soft delete. The data is not lost and an admin can reactivate it, but in production I would restrict Deactivate through command bar customization.
- Gap closed: The Issue status column already handles the process (New, Escalated, Resolved), so Deactivate is not needed in this app. I hid Deactivate on the main grid and main form command bars and retested as Staff Test: no Deactivate and no Delete in the list or on the record, and the Owner field is locked. This is a UI-level fix; Write permission still exists, so full enforcement would need server-side logic.
- Also found and fixed: Staff Test saved an issue with a blank title, because Issue title was not required. Made Issue title Business required.
- Scope change: "Staff cannot edit Issue status" would need column-level (field) security, so I simplified it to "Staff can only see and edit their own issues".
- Note: A custom role must start as a copy of Basic User, otherwise users lack the base privileges needed to open an app. Linking a lookup needs Append on one table and Append To on the other.

### Day 7: Review
- Summary of my design decisions:
    1. Checkout Issue is user-owned, so access can depend on who logged it; Machine is organisation-owned shared reference data.
    2. Machine is a lookup, not free text, because "SCO-1" vs "SCO 1" made counts per machine wrong.
    3. The Machine relationship uses Restrict delete, so machines with logged issues can't be removed and history stays intact.
    4. Always-needed fields use the column's required setting; conditional logic uses business rules. An Entity-scope error message rule blocks bad data even outside the form.
    5. Roles are copies of Basic User, because a role built from scratch can't open the app. Linking an issue to a machine needs Append on Checkout Issue and Append To on Machine.
    6. Staff could Deactivate issues with only Write permission (a soft delete); I hid Deactivate on the command bar and documented that it's a UI-level fix.
- Also did: wrote these decisions in Japanese and English for interview practice, organised screenshots with clear names, and added the Owner column to the Active Checkout Issues view so the app shows who logged each issue.

### Day 8: Automation
- Built: An automated cloud flow inside the solution, "Alert supervisor on repeated machine issues". Trigger: a Checkout Issue is added (Dataverse, Organization scope). List rows counts active, unresolved issues on the same machine with the filter _yg_machine_value eq '<Machine>' and yg_issuestatus ne 128300002 and statecode eq 0. A Condition checks whether the count equals 5, and if so, Update a row sets the new issue's status to Escalated.
- Tested: 4th open issue on SCO-1, condition false, nothing changed. 5th issue, condition true (expression result "true" in the run history). 6th issue, condition false again, so the alert fires once when the threshold is crossed, not on every new issue. Only the triggering issue was escalated; the other issues on SCO-1 stayed New.
- Broke: My first design sent an email with the Mail connector, but the run failed because Microsoft restricts that connector for new tenants. The suggested Gmail connector can't be combined with Dataverse in one flow, and Office 365 Outlook needs a mailbox my trial account doesn't have.
- Fixed: Replaced the email with Update a row, so the flow escalates the issue and supervisors see it in the Escalated issues view. Resubmitted the failed run, which then succeeded.
- Note: For Row ID I used the trigger's Checkout Issue ID, not the one from List rows; the List rows ID would have created an Apply to each loop and escalated all five issues. The flow runs with the connection owner's permissions, so it needs an account that can read all issues.

### Day 9: Reporting
- Built: Three charts on Checkout Issue (Issues by machine, Issues by status, Issues per day grouped by day) and a classic Supervisor dashboard in the app. The dashboard shows Issues by machine from the Open issues view, the other two charts from Active Checkout Issues, and a list of Escalated issues. Used a classic dashboard instead of a table "overview" dashboard, because each component needed its own view.
- Tested: As admin, the dashboard showed SCO-1 with 6 open issues, New 7 / Escalated 2 / Resolved 2, and both escalated issues, including the one escalated by the flow. SCO-4 didn't appear in Issues by machine because its only issue is resolved, which is correct for the Open issues view. As Staff Test, the same dashboard showed only Staff Test's own issue (SCO-1: 1, New: 1) and an empty Escalated list, so row-level security also applies to charts and dashboards.
- Broke: Chart names were auto-generated as "Issue title by Machine" and so on, which read badly on the dashboard. The chart designer also showed US date format, while the app showed Irish format.
- Fixed: Renamed the charts before publishing. Date format follows each user's personal setting in the app.
- Limitations noted: Days with zero issues don't appear in the per-day chart, so quiet days are hidden; a tool like Power BI would be better for vendor trend reports. Staff can open the dashboard because it's in the app navigation; in production I'd hide it from staff or use a separate staff app.

### Day 10: Data import
- Built: Imported 42 sample issues (1 Oct to 10 Nov) with the app's Import from CSV. Turned the alert flow off first, so a bulk load wouldn't trigger it 42 times. Dates were written year-first (2026-10-02 13:45) to avoid the day/month confusion from Day 4. Checked the mapping before importing: each Error type value matched its choice, and the Machine lookup matched by Machine code. Result: 42 successes, 53 issues in total, and the Supervisor dashboard now shows open issues concentrated on SCO-3 and SCO-4 (the v2.4 machines).
- Tested with 3 deliberately bad rows: SCO-5 failed ("lookup reference could not be resolved"); a Resolved issue with no note was blocked by Rule 2 on the server ("ISV code aborted the operation"); a row with an invalid Error type ("Card broken") was imported with Error type empty.
- Broke: "Import from Excel" rejected a normal .xlsx because it expects a file exported from Dynamics. In the bad-rows mapping, the wizard refused to auto-map the Error type column and offered to map "Card broken" as a value, which risked adding it as a new choice. Most importantly, a column set to Business required is only enforced by forms, not by imports, so the empty Error type got through.
- Fixed: Used Import from CSV instead. Set "Card broken" to Ignore rather than mapping it to a real value. Deleted the invalid row and added Rule 3 ("Require error type", Entity scope, message "Select an error type."). Re-imported: all 3 bad rows now fail, each with its own reason, including my custom messages.
- Note: Required in the form doesn't mean required in the data. Server-side rules (Entity-scope business rules) protect data from imports and integrations; turn automation off during bulk loads, and read the mapping instead of clicking through.

### Day 11: Admin tasks and solutions
- Built: Turned on auditing for the environment (the table already had auditing on from Day 2). Created a test issue and changed its status; Audit History showed the Create event with every initial value, then each Update with Changed By, date, Old Value and New Value (Issue status: New to Escalated). Ran the Solution checker: 0 issues (Critical, High, Medium and Low all 0). Updated the solution version to 1.1.0.0, published, and exported both a managed and an unmanaged package as build 1.1.0.1.
- Checked: The export listed three Microsoft managed-solution dependencies (app framework, system app actions, app module assets); import fails if a target environment lacks them. Opened the exported customizations.xml to confirm it contains both tables, four choices, two security roles, three business rules and the flow, and that the invalid "Card broken" value from Day 10 was never added to Error type.
- Broke: The solution still contained a Mail connection reference left over from the original email design on Day 8. The flow no longer used it, but importing the package elsewhere would have required a Mail connection, which is restricted on new tenants. The Overview also warned that 2 environment variables needed updating.
- Fixed: Removed the Mail connection reference from the solution (not from the environment) before exporting, and verified it's absent from the exported file. The environment variables belong to Microsoft's built-in Customer Service components, so I left them unchanged.
- Note: Audit only records changes made after it's turned on, and must be enabled at environment, table and column level. Both export packages are saved outside the trial as a backup.

### Day 12: UAT
- Test cases run: 14, each linked to a user story (staff logging, supervisor escalation, manager reporting), run on version 1.1.0.1 with two windows (admin and Staff Test) and a screenshot per test.
- Passed / failed: Final result 12 Pass, 2 Pass with note, 0 Fail. Four tests passed only on retest (TC02, TC04, TC11, TC14).
- Defect found (DEF-01): Issue date wasn't required, so a staff issue was saved without a date and appeared as "(blank)" in the per-day chart. Fixed with a fourth business rule, "Require issue date" (Entity scope), gave the old record a date, and retested.
- Defect I introduced (DEF-02): Trying to replace the standard "Required fields" message with my own, I added a Show Error Message action to Rule 1's Resolved branch. It didn't change the message, and it would have blocked every resolve, even with a note. Reverted it and retested TC04. Lesson: don't change the design during a test run, and accept a correct behaviour even if the wording isn't mine.
- Retests: TC11 rerun with a 4-row file and Error type mapped manually (invalid value set to Ignore); all 4 rows failed for their own reason, including my custom messages for Rules 3 and 4. TC14 retaken to show the audit Update row (New to Escalated).
- Released: Exported version 1.1.0.2 as managed and unmanaged packages, with the solution checker run on export (0 issues). The package also contains Dataverse search index components (DVTableSearch) added automatically by the platform; I checked that they don't add any new managed dependencies (still the same three Microsoft solutions as 1.1.0.1), so I kept them.
- Note: A UAT sheet with honest notes and retests is more credible than a clean 14/14.


