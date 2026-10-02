# Accessibility test script

| | |
|---|---|
| **Document** | 24 · Accessibility test script |
| **Version** | 0.1 |
| **Status** | Draft for review |
| **Date** | 29 September 2026 |
| **Owner** | Peter Pirisola |
| **Scope** | Product (client-neutral). Client specifics are kept in client notes. |
| **Classification** | Public copy (see the note below the table) |
| **Audience** | The release owner and testers |
| **Related documents** | Non-functional requirements (NFR-ACC) · User flows · Design system · Development standards §14 and §19 · [Release test script](22-release-test-script.md) · ADR-22 |

> **Public copy.** Published from D8A AI XYZ's internal document set on 2 October 2026. References to other numbered documents, architecture decision records (ADRs) and reviews point at the internal specification, which is not published; they are shown as plain text here.

## Purpose

This is the manual accessibility pass run for every release candidate (doc 18 §14, accessibility row). The automated tests check names, toggle states and live-region text in code; this script checks what only a real screen reader, a real keyboard and real Windows display settings can show. It is run by hand on one Windows 11 PC with the release candidate installed. Its results table (section 8) is attached to the release record and to group G of the [Release test script](22-release-test-script.md).

The expected announcements are copied word for word from NFR-ACC in document 05, or, where NFR-ACC names a message without quoting it (the estimate line, the nudge, a name error, the loading and toast texts, the diagnostics texts), from the message text in document 04 and the built product. A screen reader adds words of its own: a role ("button", "tree item", "check box"), a position ("2 of 5") or a state ("selected"). That is expected; the words quoted here must be present, in this order. Where the build says something else, record exactly what was heard and raise an issue.

Fill in the **Result** column with Pass, Fail (and the issue number) or N/A (with the reason). A failure is classed as:

- **Critical:** a function cannot be used at all by keyboard or with a screen reader (for example a keyboard trap, a button that cannot be reached, a window whose content is not read).
- **Serious:** a function can be used only with a workaround, or a Must requirement in NFR-ACC is not met (for example a missing announcement, a focus indicator that disappears, contrast below the target).
- **Moderate or minor:** everything else (wording, the order of words, an extra announcement).

The exit criterion is doc 18 §14's: **no critical or serious issues open at release**.

## 1. Set-up

| Item | What is needed |
|---|---|
| PC | Windows 11 x64, one monitor at 1920 × 1080. For section 6, the same monitor at 150 % and 200 % (Settings › System › Display › Scale). |
| Excel | Excel for Microsoft 365 (Microsoft 365 Apps for enterprise), 64-bit, Current Channel. |
| The add-in | The release candidate under test, installed from its MSI as in step A3 of the [Release test script](22-release-test-script.md), with `config.json` pointing at the D8A test tenant. Not a Debug build. |
| Model | The sample model Inwards Analytics in the test workspace, with its perspectives and display folders (Claim Transactions › Claims with 8 fields, Counts, Recoveries). |
| Accounts, by role | **Full access** (Viewer + Build, no RLS role) for every step; **OLS** (an object-level security role that hides at least one column) for the Missing fields steps. The native-query approval setting is turned off for both (decision D-15). |
| Configuration copy | For the compact confirmation (P12) and the nudge (N10): a copy of the test configuration with `nudgeRowThreshold` and `maxRowsBeforeWarning` set low (for example 100 and 1,000), as the UI smoke test S14 does. Put the normal file back afterwards. |
| Tools | [Accessibility Insights for Windows](https://accessibilityinsights.io/downloads/), current version (FastPass and the Colour Contrast Analyzer). Narrator (built in; Ctrl + Windows + Enter turns it on and off). [NVDA](https://www.nvaccess.org/download/), current version, with Tools › Speech Viewer open so the exact words can be copied into the Result column. JAWS, current version, **when a licence is available**: run section 4 with JAWS and record it as its own row in section 8. |
| Display settings | Scaling at 100 % for sections 2 to 5, then 150 % and 200 % in section 6. Contrast themes (Settings › Accessibility › Contrast themes): **Desert** and **Night sky** in section 6. Animation effects (Settings › Accessibility › Visual effects): **on** for sections 2 to 6, then off and on in section 7. |
| First-use notice | To see the notice again, close Excel and delete `%LOCALAPPDATA%\D8A\DataPlatform\settings.json` (it holds only the per-user flags). |
| Evidence | For each failure: a screenshot, the Speech Viewer text or a note of what Narrator said, and the FastPass results file. Keep them with the release record. |

## 2. Automated pass

Open each window, select it in Accessibility Insights for Windows (the whole window, with its descendants) and run **FastPass**. Record the number of failures; save the results file for any failure.

| # | Window | How to open it | Expected | Result |
|---|---|---|---|---|
| P1 | New Table, step 1 Workspace | New Table | FastPass: 0 failures | |
| P2 | New Table, step 2 Model | Choose a workspace | FastPass: 0 failures | |
| P3 | New Table, step 3 Perspective | Choose Inwards Analytics | FastPass: 0 failures | |
| P4 | New Table, step 4 Fields, Table layout, with Claim Transactions › Claims open | Choose Claims | FastPass: 0 failures | |
| P5 | New Table, step 4 Fields, Matrix layout, with a field in each area | Switch to Matrix | FastPass: 0 failures | |
| P6 | New Table, step 5 Filters, with one filter card applied and one not applied | Next | FastPass: 0 failures | |
| P7 | New Table, step 6 Load, with the estimate shown | Next | FastPass: 0 failures | |
| P8 | Edit, on a loaded table (step 4) | Select a cell in the table, Edit Table | FastPass: 0 failures | |
| P9 | The loading overlay | Load a large table and run FastPass while it loads | FastPass: 0 failures | |
| P10 | The error dialog with **Show details** open | Disconnect the network, New Table (DP-E10), Show details | FastPass: 0 failures | |
| P11 | The Delete confirmation | Delete Table | FastPass: 0 failures | |
| P12 | The compact confirmation ("Load about … rows?") | With the low-threshold configuration copy, set a sheet filter to (All) on a table above `maxRowsBeforeWarning` | FastPass: 0 failures | |
| P13 | The closing confirmation | New Table, choose a workspace and a model, press Esc | FastPass: 0 failures | |
| P14 | Missing fields | As the OLS account, refresh a workbook saved by the full-access account with the hidden column in a table | FastPass: 0 failures | |
| P15 | About | Account menu › About Data Platform | FastPass: 0 failures | |
| P16 | The text report (Diagnostics check) | Account menu › Run diagnostics check | FastPass: 0 failures | |
| P17 | The sheet value dropdown, in list mode and in search mode | Select a filter value cell | FastPass: 0 failures | |
| P18 | The notice pane above the sheet | As a new user (settings file deleted), open a workbook last refreshed by the other account | FastPass: 0 failures | |
| P19 | The Approved Workspaces view | Approved Workspaces | FastPass: 0 failures | |
| P20 | A loaded sheet (NFR-ACC-14) | Excel's Review › Check Accessibility on a sheet the add-in created, with a table and its filter area | Excel reports 0 errors | |

## 3. Keyboard-only

Put the mouse out of reach. Each run follows a user flow in document 09. In every run and every window, check the behaviour that NFR-ACC-01, NFR-ACC-02, NFR-ACC-03 and NFR-ACC-07 require:

- every control is reached with Tab or the arrow keys, in the visual order, and nothing traps the focus;
- the focus is always visible: a 2 px Violet Ink outline with a 2 px offset, never hidden behind other content;
- Esc cancels or closes the window or dropdown that has the focus;
- every ribbon command has a key tip.

A note on the focus ring. The ring is the kit's (document 25 §8.3): 2 px Violet Ink with a 2 px offset, on buttons, fields, cards, chips, the query tabs and the tree and list rows (radius 6 on rows, full on chips). WPF draws it only once the keyboard has been used in that window: a dialog opened from the ribbon with the mouse shows no ring on its default button until the first key press, and the ring then appears where the focus already is (UI comparison C0-8). That is expected, not a failure. In K10, check whether the ring is on **Cancel** the moment the loading panel appears or only after a key: the focus is placed there by code, not by the keyboard, so record which you see; a ring that appears after one key press is a Minor result, a focus that is not on Cancel is Critical.

| # | Run | Steps | Expected | Result |
|---|---|---|---|---|
| K1 | F-02, New Table to a loaded table | 1. Alt, G, N. 2. On step 1, Tab to the cards; the arrow keys move between cards without choosing; Enter chooses "Inwards Reinsurance - Production" (or the test workspace). 3. On step 2, Tab to the Inwards Analytics card, Space. 4. On step 3, Tab to Claims, Enter. 5. On step 4, Tab to the field tree; Enter on Claim Transactions and on Claims opens them; Space ticks Paid Claims; tick Line of Business and Underwriting Year the same way. 6. In "Your table", Tab to Paid Claims; Alt + ↑ moves it up; the ↑ ↓ × buttons do the same; Delete removes a field (tick it again). 7. Enter (Next). 8. On step 5, add a filter on Underwriting Year: Tab to the column in the tree, Enter, Space on two chips, Apply. 9. Enter (Next). 10. On step 6, Tab through the name, the destination, the summary and Show the query; Enter loads. | Steps 1 to 3 have no Next: Enter or Space on a card chooses it and moves on. On arriving at steps 2 and 3 the focus is on the lead text and no card looks chosen. Opening or closing a folder never moves the focus off it (NFR-ACC-18). The table loads with no mouse used. | |
| K2 | F-02, going back | From step 4 of K1, Shift + Tab to the step bar and choose Perspective, then Model. | The earlier choice is shown selected and has the focus (NFR-ACC-01). | |
| K3 | F-13, the matrix (NFR-ACC-17) | Steps 1 to 3 as K1. On step 4 tick Line of Business, Underwriting Year, Paid Claims and Outstanding Reserves. Tab to the Table \| Matrix switch, → to Matrix, Enter or Space. Tab to "Move Underwriting Year to Columns", Enter. Use ↑ ↓ within Rows, and × to remove a field (then tick it again). Next, Next, Load table. | Every placement is made without dragging; the matrix loads as a PivotTable with Underwriting Year across the top. | |
| K4 | F-06, Edit and a perspective change | Select a cell of the K1 table with the arrow keys; Alt, G, E. Go to step 3 with the step bar and choose Premium. | The inline question ("… are not in Premium and will be removed: …") appears with "Keep Claims" and "Change to Premium", the focus on the primary button; Tab and Enter operate both. Update table works from the keyboard. | |
| K5 | F-08, Refresh | Alt, G, R on a table; then Alt, G, A. As the OLS account, Refresh All on the workbook of release test script step E2. | Both refresh. The Missing fields dialog opens with the focus on **Stop**; Tab cycles the field list (when it scrolls), Stop, Continue and close; with the list focused, the arrow keys and Page Up/Down scroll it; Esc acts as Stop. | |
| K6 | F-07, Delete with the confirmation | Select a cell of a table; Alt, G, D. | The confirmation opens with the focus on **Cancel** (D-05); Tab reaches Delete; Enter on Cancel keeps the table; Enter on Delete removes it. | |
| K7 | F-04, the sheet value dropdown | Move to a filter value cell with the arrow keys. In the dropdown, type in the search box, Tab to the values, Space ticks one, Tab to Apply, Enter. Open it again and press Esc. | The dropdown opens when the selection lands on the cell (D-16), with the focus in the search box; Apply refreshes only that table; Esc closes it and the value is unchanged. | |
| K8 | F-09, Approved Workspaces | Alt, G, W. Type in the search box; Tab to a card's "New table from here →"; Enter. | New Table opens at step 2 with that workspace chosen and no model chosen. | |
| K9 | The Account split button and its menu | 1. Alt, G: note the key tip on Account. 2. Press S. 3. Alt, G again, move to Account with the arrow keys and press ↓ (or Alt + ↓) to open the menu. 4. In the menu, ↑ ↓ move between the items; try each key tip (O Sign out, B About Data Platform, C Copy diagnostics, K Run diagnostics check). 5. Repeat signed out. | S runs the primary action: Sign in when signed out, About when signed in. Record how Office shows the menu half (a key tip of its own, or ↓). Each menu item is reached and runs from the keyboard; Sign out is listed only when signed in. In the text report the focus starts on **Close**; Tab reaches the report text and **Copy report**; Esc, and Enter on Close, close it. | |
| K10 | Cancel on a long load (D-07) | Start a long load. Without using the mouse, press Esc; start it again and press Space (the focus is on **Cancel**). Repeat with the overlay of Run diagnostics check. When the overlay has closed, type in a cell. | Cancel is reachable: the overlay takes focus on Cancel, Esc cancels (NFR-ACC-01, D-07). Cancel shows the focus ring as soon as the overlay appears; Esc and Space each stop the operation once: the detail line then reads "Cancelling… waiting for Excel" and Cancel is greyed until the operation ends, a second Esc doing nothing; nothing is written, and the toast says "Cancelled · nothing was changed". After the overlay closes the keyboard is back in Excel. An overlay without Cancel (none today) would not take the focus from Excel. A missing route is a **Critical** result. | |
| K11 | The closing confirmation (D-07) | New Table, choose a workspace and a model, press Esc. | A confirmation asks before the choices are lost; its buttons are reached with Tab; going back keeps the choices; confirming closes the window with nothing written. | |
| K12 | F-15, the first-use notice | As a new user, New Table. Shift + Tab from the lead text to the notice and "Got it"; Enter. Then, with the settings file deleted again, open a workbook last refreshed by the other account and press F6 until the notice pane above the sheet has the focus; Tab to "Got it"; Enter. | The notice is at the top of the Tab order; "Got it" works with Enter or Space; afterwards the focus is on the step's lead text (in the window) or back where it was (above the sheet). | |
| K13 | F-10, an error | Disconnect the network; New Table. Tab to Show details, Enter; Tab to Copy details, Enter; reconnect; Tab to Try again where it is offered, Enter. | Every button is reached and works; Esc closes the dialog. | |
| K14 | F-01, sign in | Signed out: Alt, G, S. | The Windows account picker opens and is operated by keyboard; afterwards the Account label reads "Signed in as …". | |

## 4. Narrator script

Turn on Narrator (Ctrl + Windows + Enter) with its default settings. Move with Tab and the arrow keys as in section 3 and listen. Quote in the Result column anything that differs.

| # | Step | Expected announcement (verbatim) | Requirement | Result |
|---|---|---|---|---|
| N1 | New Table. Listen to step 1. | The count line "`<n>` approved" (for a client with 16 accessible approved workspaces, "16 approved"; the test tenant gives its own count). Each card is read as its name plus summary, for example "Inwards Reinsurance - Production, Production, 3 models". | NFR-ACC-04, NFR-ACC-05 | |
| N2 | Click, or press Enter on, a workspace with one model ("Pricing - Production" where the test tenant has one; otherwise any workspace). | "Step 2 of 6, Model." The model card is not announced as selected, and the focus is on the step's lead text. | NFR-ACC-16 | |
| N3 | Press Back. | "Step 1 of 6, Workspace." | NFR-ACC-16 | |
| N4 | Choose Inwards Analytics, then the Claims perspective. On step 4, open Claim Transactions, open Claims, tick Paid Claims. | "Claims, 1 selected of 8, expanded". Counts and Recoveries are announced as collapsed. The nesting level is announced, so the tree is not read as a flat list. | NFR-ACC-18 | |
| N5 | Switch to Matrix. | "Layout Matrix: Rows, Columns and Values". The switch's pressed state is announced. | NFR-ACC-17 | |
| N6 | Tab to the ⇄ button of Underwriting Year and press Enter. | The button is read as "Move Underwriting Year to Columns"; after Enter, "Underwriting Year moved to Columns". | NFR-ACC-17 | |
| N7 | Switch back to Table. | "Layout Table" | NFR-ACC-17 | |
| N8 | On step 5, add a filter on a column in search mode (more than 1,000 values) and type a few letters. Then leave one filter card without values. | The match count "`<m>` matches" ("50 matches" when 50 or more match). The cards read "Applied" and "Not applied" as text. | NFR-ACC-05, NFR-ACC-04 | |
| N9 | Go to step 6. | The estimate line is announced when it appears and when it changes, without the focus moving: "Estimating", then "About `<N>`, as your access allows" (or "Could not estimate"). | NFR-ACC-20 | |
| N10 | With the low-threshold configuration copy and no filter, go to step 6. | The nudge is announced when it appears, for example "About 180,000 rows. A filter such as Underwriting Year or Month makes the table quicker to load and refresh. You can load it as it is."; "Add a filter" is read as a button. Above 1,048,576 rows the Excel-limit message is announced the same way. | NFR-ACC-20 | |
| N11 | On step 6, type the name of a table that already exists in the workbook. | The error is announced while the focus stays in the name box: "A table or query called `<name>` already exists in this workbook. Choose another name." | NFR-ACC-05, NFR-ACC-12 | |
| N12 | Load table. | The start and the finish of the load are announced: the loading overlay's "Querying Inwards Analytics as you", then the toast "Loaded `<name>` · `<n>` rows". | NFR-ACC-05 | |
| N13 | As a new user (settings file deleted), open New Table. | The notice is announced once, without the focus moving: "Some columns and rows may not be available to you because of row-level security and object-level security." Tab reaches "Got it", read as a button. The icon is either not read or read as "Information". | NFR-ACC-19 | |
| N14 | Account menu › Run diagnostics check, signed in. When the report opens, Tab to the report text and read it line by line with the arrow keys, as a document. | The text box is read as "Diagnostics check"; each line is read with its word first, for example "PASS Configuration: …", "PASS Sign-in: Signed in as …", "PASS REST access: …", "PASS XMLA access: ok in … ms (…)". | NFR-ACC-04 | |
| N15 | Tab to **Copy** and press Enter. | "Copied" is announced (the button also reads "Copied" for 2 seconds), without the focus moving. | NFR-ACC-05 | |
| N16 | Account menu › Copy diagnostics. | The toast is read: "Diagnostics saved to `<path>` · the path is on your clipboard". | NFR-ACC-05 | |

## 5. NVDA script

Run the same steps with NVDA (current version) and Narrator turned off. Keep Tools › Speech Viewer open and copy what it shows into the Result column when it differs. Use focus mode in the windows (NVDA + Space switches modes); NVDA + ↓ reads from the cursor to the end.

| # | Step | Expected announcement (verbatim) | Result |
|---|---|---|---|
| V1 | As N1 | "`<n>` approved" ("16 approved" for 16 workspaces); each card as its name plus summary | |
| V2 | As N2 | "Step 2 of 6, Model."; the model card not announced as selected | |
| V3 | As N3 | "Step 1 of 6, Workspace." | |
| V4 | As N4 | "Claims, 1 selected of 8, expanded"; Counts and Recoveries collapsed; the tree level announced | |
| V5 | As N5 | "Layout Matrix: Rows, Columns and Values" | |
| V6 | As N6 | "Move Underwriting Year to Columns", then "Underwriting Year moved to Columns" | |
| V7 | As N7 | "Layout Table" | |
| V8 | As N8 | "`<m>` matches" ("50 matches"); "Applied" and "Not applied" as text | |
| V9 | As N9 | "Estimating", then "About `<N>`, as your access allows" | |
| V10 | As N10 | The nudge text, "Add a filter" as a button, and the Excel-limit message | |
| V11 | As N11 | "A table or query called `<name>` already exists in this workbook. Choose another name.", with the focus still in the name box | |
| V12 | As N12 | The load's start and finish | |
| V13 | As N13 | "Some columns and rows may not be available to you because of row-level security and object-level security." once; "Got it" as a button | |
| V14 | As N14; read the report with NVDA + ↓ and with ↑ ↓ | Every line, with its PASS, FAIL or SKIP word | |
| V15 | As N15 | "Copied" | |
| V16 | As N16 | "Diagnostics saved to `<path>` · the path is on your clipboard" | |

## 6. Contrast and scaling

| # | Step | Expected | Requirement | Result |
|---|---|---|---|---|
| C1 | Turn on the **Desert** contrast theme. Restart Excel. Open every window of section 2 (P1 to P19). | The windows use the system colours: all text, borders, the control boundaries (text inputs, the search field, dropdowns, the sheet filter value cell), the focus outline and the icons stay visible; nothing is drawn in a fixed brand colour that the theme makes unreadable. | NFR-ACC-09, NFR-ACC-07 | |
| C2 | The same with **Night sky**. | As C1. | NFR-ACC-09 | |
| C3 | Before the first release, and whenever a window's template or the design tokens change, the same with **Aquatic** and **Dusk** (NFR-ACC-09 names every built-in theme). | As C1. | NFR-ACC-09 | |
| C4 | Contrast theme off. With the Colour Contrast Analyzer of Accessibility Insights, measure the boundary of a text input and of the sheet filter value cell against its background. | The control-boundary token #7E838E: at least 3:1 (3.80:1 on white, 3.64:1 on Snow, 3.46:1 beside a banded row). Card, pane, dialog and notice edges (Silver) are decorative and outside the rule (D-03). | NFR-ACC-06 | |
| C5 | Measure the body text, the secondary text, the focus outline, the "Applied" pill, the nudge and the Excel-limit message. | Normal text at least 4.5:1, large text at least 3:1, the focus outline and meaningful icons at least 3:1, as in the contrast table of document 05. | NFR-ACC-06 | |
| C6 | Tab through each window of section 2. | The focus outline is visible on every focusable element, in the default theme and in C1 to C3. | NFR-ACC-07 | |
| C7 | Scale to **150 %**. Restart Excel. Open every window of section 2 and run K1 and K3. | No content or function is lost; panes scroll inside the window; nothing is clipped; the text report wraps. | NFR-ACC-13, NFR-COMP-05 | |
| C8 | Scale to **200 %** on the 1920 × 1080 monitor. Repeat C7. | As C7; the window shrinks to the work area (at least 800 × 500 DIP) and its panes scroll; the notice text wraps and "Got it" stays visible. | NFR-ACC-13, NFR-ACC-19 | |
| C9 | At 100 % scaling, set Windows text size to 150 % (Settings › Accessibility › Text size). Repeat C7. | As C7. | NFR-ACC-13 | |

## 7. Motion

| # | Step | Expected | Requirement | Result |
|---|---|---|---|---|
| M1 | Animation effects **on**. Load a table and watch the loading overlay; move between steps. | The violet segment travels round the D8A mark in a 2,600 ms loop; step and panel transitions use the brand durations (120, 220 or 400 ms). Nothing flashes. | NFR-ACC-08 | |
| M2 | Animation effects **off** (Settings › Accessibility › Visual effects). Restart Excel. Repeat M1. | The loading segment holds still; transitions are removed; the load still completes, and its start and finish are still announced. | NFR-ACC-08 | |

## 8. Results

One table per release candidate, attached to the release record and referred to from group G of the [Release test script](22-release-test-script.md). Copy the table below for each release.

Release: `data-platform/v<version>`, `BuildVersion` __________ · Tester: __________ · Date: __________

| Item | Tool | Result | Issue link |
|---|---|---|---|
| P1 to P19 | Accessibility Insights for Windows, FastPass | | |
| P20 | Excel Check Accessibility | | |
| K1 to K14 | Keyboard | | |
| N1 to N16 | Narrator | | |
| V1 to V16 | NVDA | | |
| N1 to N16 | JAWS (when a licence is available) | | |
| C1 to C3 | Windows contrast themes | | |
| C4 to C6 | Colour Contrast Analyzer and visual check | | |
| C7 to C9 | Scaling and text size | | |
| M1, M2 | Animation effects | | |
| **Exit criterion** | **No critical or serious issues open at release** (doc 18 §14) | | |

**Closed risk.** The loading overlay (loads, refreshes and Run diagnostics check) used to be shown without taking the keyboard focus, so Cancel had no keyboard route. Since plan 13 an overlay with Cancel opens activated with the focus on Cancel, Esc cancels, and Excel gets the focus back when it closes (K10, OQ-04 closed).

Release owner: ____________________  Date: __________

## Open questions

| ID | Question | Needed from | Needed by |
|---|---|---|---|
| OQ-01 | How does Office expose the menu half of the Account split button by keyboard: a key tip of its own, or ↓ on the focused button (K9)? The answer goes into the quick guide. | Release owner (first run of K9) | Before the pilot |
| OQ-02 | Does the test tenant have a one-model workspace for N2 and V2 ("Pricing - Production" in the examples)? Without one, N2 runs on a workspace with several models and the "not selected" check still applies. | Peter Pirisola (D8A test tenant) | Before the first run |
| OQ-03 | Is a JAWS licence available for the pilot release, or does the pilot accept Narrator and NVDA only (NFR-ACC-04 names all three)? | Peter Pirisola | Before the pilot |
| OQ-04 | **Closed 30 September 2026 (plan 13):** the loading overlay with Cancel takes the focus on Cancel and Esc cancels; K10 checks it. | Release owner (first run of K10) | Before the pilot |

## Change history

| Version | Date | Author | Change |
|---|---|---|---|
| 0.1 | 29 September 2026 | Peter Pirisola | First draft, for workstream 8 part 2 (ADR-22): the automated pass, the keyboard-only runs per user flow, the Narrator and NVDA scripts with the NFR-ACC announcements, contrast and scaling, motion and the results table. |
| 0.2 | 29 September 2026 | Peter Pirisola | Review fixes: K10 tries Esc and Tab to reach the overlay's Cancel, and a missing route is Critical (known risk, OQ-04); where the expected texts come from. |
| 0.3 | 30 September 2026 | Peter Pirisola | Plan 13 (design kit, task 6): the overlay with Cancel takes the focus on Cancel and Esc cancels; K10 expects that route and Excel's focus back afterwards; the known risk and OQ-04 closed. |
| 0.4 | 30 September 2026 | Peter Pirisola | Plan 13 (ADR-24): §3 notes when WPF draws the focus ring (after the first key press) and what K10 records about the ring on Cancel; K9 names the text report's button **Copy report**. |
| 0.5 | 1 October 2026 | Peter Pirisola | Plan 14, the whole-branch review's fix round: K10 expects the cancelling state ("Cancelling… waiting for Excel", Cancel greyed until the operation ends). |
