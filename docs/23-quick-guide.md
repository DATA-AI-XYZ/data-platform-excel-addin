# Quick guide

| | |
|---|---|
| **Document** | 23 · Quick guide |
| **Version** | 0.1 |
| **Status** | Draft for review |
| **Date** | 29 September 2026 |
| **Owner** | Peter Pirisola |
| **Scope** | Product (client-neutral). Client specifics are kept in client notes. |
| **Classification** | Public copy (see the note below the table) |
| **Audience** | Users and trainers |

> **Public copy.** Published from D8A AI XYZ's internal document set on 2 October 2026. References to other numbered documents, architecture decision records (ADRs) and reviews point at the internal specification, which is not published; they are shown as plain text here.

The users' handout is this page without the table above, the open questions and the change history.

## What Data Platform does

Data Platform puts data from your organisation's approved Power BI models into Excel. You choose the fields you need and get a normal Excel table, or a PivotTable (the Matrix layout), that you can refresh at any time. Every load and refresh runs **as you**: you see only what your own Power BI access allows.

The examples use the sample model Inwards Analytics in the workspace "Inwards Reinsurance - Production".

## The Data Platform tab

| Button | What it does |
|---|---|
| **New Table** | Builds a new table or matrix in six steps |
| **Edit Table** | Reopens the steps for the selected table, with your choices |
| **Delete Table** | Removes the selected table and its setup |
| **Refresh Table** | Refreshes the selected table |
| **Refresh All** | Refreshes every Data Platform table in the workbook |
| **Approved Workspaces** | Lists the workspaces your organisation has approved for Data Platform |
| **Account** | Signs you in; once signed in, shows who you are and the version. Its arrow opens a menu: **Sign out**, **About Data Platform**, **Copy diagnostics** and **Run diagnostics check** |

## New Table in six steps

1. **Workspace.** Pick an approved workspace you have access to.
2. **Model.** Pick a semantic model, for example Inwards Analytics.
3. **Perspective.** Pick a perspective (a focused view of the model) or All Model.
4. **Fields.** Tick the columns and measures you want, from the model's folders. Choose **Table** or **Matrix** at the top.
5. **Filters.** Optional: narrow the rows, for example Underwriting Year = 2025. Filters are not required, but a large table without one gets a suggestion.
6. **Load.** Name the table, check the summary and the row estimate, and press **Load table**.

The table lands on a new sheet (or the current one) with its filters above it. A long load can be cancelled; nothing is left half-done.

## Refreshing

- **Refresh Table** refreshes the selected table; **Refresh All** refreshes them all, one after another.
- **Data › Refresh All** (Excel's own button) also refreshes Data Platform tables, even on a PC without the add-in.
- When a workbook opens with the add-in, its Data Platform tables refresh once, as the person opening it.
- If a field you used has been removed from the model, or you cannot see it, **Missing fields** asks first: **Stop** keeps everything as it is; **Continue** drops those fields and refreshes the rest.

## Editing and deleting

Select a cell in the table, then **Edit Table** (or double-click a cell of a table; in a matrix use **Edit Table** or **± Fields**) to change fields, filters or the name, and press **Update table**. **Delete Table** asks first and removes the table, its filters and its setup.

## The filter cells on the sheet

Above each table are its filters, one per row. Select a filter's value cell (or click its ▾) to open a list of values; pick values and press **Apply**. Only that table is refreshed, through the same checks as a refresh.

## What "as you" means

- You sign in with your work account. Your Power BI permissions apply to every load and refresh.
- **Row-level security** may show you fewer rows than a colleague sees; **object-level security** may hide columns from you. The first time you use Data Platform, a notice says so.
- Someone else opening your workbook sees their own rows when it refreshes, not yours.

## The first time a table loads

Excel may ask how to connect to Power BI. Choose **Organisational account**, sign in with your work account, and press **Connect**. Excel remembers it after that. Data Platform shows a note about this before your first load.

If Excel asks you to approve a "native database query" at every load, turn off Data › Get Data › Query Options › Global › Security › "Require user approval for new native database queries".

> **Two things to know**
>
> - **Sharing a workbook shares the values saved in it until someone refreshes it.** This includes a matrix: without Data Platform, a matrix does not refresh when the workbook opens and shows its saved values until someone refreshes it.
> - **Excel's own table filters hide rows on your screen; they are not a security control.**

## Something went wrong

Every message has a code, such as **DP-E10**.

- **Try again** appears when the problem may pass (the network, a busy model); it runs the same command again.
- **Show details** shows the code, the time and the IDs the service desk needs; **Copy details** puts them on the clipboard.
- Contact your IT service desk with the copied details. If they ask for diagnostics: open the **Account** menu (the arrow under Account), choose **Copy diagnostics**, and attach the file whose path is now on your clipboard. You do not need to be signed in.
- If they ask for a diagnostics check: choose **Run diagnostics check** from the same menu, press **Copy report** in the window that opens, and paste the result into your reply.

## Where it works

Excel for Microsoft 365 on Windows (the desktop app). Excel for the web, Mac and mobile open the workbooks and show the values last saved; they cannot refresh through Data Platform.

## Logs

Data Platform keeps its log on your PC in `%LOCALAPPDATA%\D8A\DataPlatform\logs`: the last 14 log files (about two weeks). It holds no data values. Copy diagnostics saves the last 7 days of it, with the setup log (`register.log`), in one file for the service desk, named `diagnostics-` and the date and time to the second.

## Open questions

| ID | Question | Needed from | Needed by |
|---|---|---|---|
| OQ-01 | Should the client's own name for the tab (`ribbonLabel`) replace "Data Platform" in the copy handed to its users? | Client trainers | Before the pilot |

## Change history

| Version | Date | Author | Change |
|---|---|---|---|
| 0.1 | 29 September 2026 | Peter Pirisola | First draft, with the two statements release criterion 8 requires. |
| 0.2 | 29 September 2026 | Peter Pirisola | Copy diagnostics after sign-in; the handout omits the header table, the open questions and the change history. |
| 0.3 | 29 September 2026 | Peter Pirisola | The log keeps the last 14 files (about two weeks); what Copy diagnostics saves. |
| 0.4 | 29 September 2026 | Peter Pirisola | The Account menu: Copy diagnostics without signing in, and Run diagnostics check (ADR-22). |
| 0.5 | 30 September 2026 | Peter Pirisola | The diagnostics check's button is **Copy report** (plan 13, ADR-24). |
