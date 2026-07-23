# CyberArk Privilege Cloud — Power BI Dashboard Guide (V7)

**Companion to:** `Antigravity_Dashboard_Script_V6.ps1` (the script is unchanged from V6 —
this is a guide revision with corrected DAX and much more detailed page-build steps).

This guide takes you from the 14 CSVs in the `Latest/` folder to a finished 7-page report,
**assuming no prior Power BI experience**. Follow it top to bottom.

> [!IMPORTANT]
> **About TRUE/FALSE in DAX.** The script writes boolean columns (e.g. `IsDefaultSafe`,
> `AutoManaged`, `HasCPM`, `IsEmpty`, `NamingCompliant`, `IsStale`, `RotationOverdue`,
> `IsComponent`, `IsLoggedOn`) into the CSVs as **text** (`True`/`False` or `TRUE`/`FALSE`).
> When Power BI imports them as a **Text** column, you must compare them as quoted strings:
> `= "TRUE"` / `= "FALSE"`, **not** `= TRUE` / `= FALSE`. All measures below already do this.
> DAX text comparisons are **case-insensitive**, so `"TRUE"` matches `True`, `true`, or `TRUE`.
> If a measure ever returns 0/blank unexpectedly, click the column in Data view and confirm
> the actual text (some columns show `True`, some `TRUE` — both still match).

---

## Data Sources (14 CSVs)

| Dimension tables | Fact tables |
|---|---|
| `DIM_Date` | `FACT_Accounts` |
| `DIM_Region` | `FACT_Users` |
| `DIM_Platform` | `FACT_Groups` |
| `DIM_Safe` | `FACT_CPMFailed` |
| | `FACT_PlatformSummary` |
| | `FACT_VaultConnectivity` |
| | `FACT_SafeMembers` *(opt-in)* |
| | `FACT_AdminAudit` *(opt-in)* |
| | `FACT_OnboardingTrend` |
| | `FACT_MetricsSummary` |

> `FACT_SafeMembers` and `FACT_AdminAudit` only contain rows when the script runs with
> `$CollectSafeMembers = $true`. Otherwise they are header-only CSVs (Page 8 will be empty
> but the model still loads).

---

## Step 1: Import Data

1. Open **Power BI Desktop** → **Home** ribbon → **Get data** → **Text/CSV**.
2. Browse to your `Latest/` folder and import each CSV (or use **Get data → Folder**,
   point at `Latest/`, then **Combine & Transform**).
3. For each query in **Transform data** (Power Query Editor), set column types:
   - Date columns (`DateKey`, `CreatedDate`, `LastChangeDate`, `LastVerifyDate`,
     `LastLoginDate`, `LastLogonDate`, …) → **Date** (or Date/Time).
   - Count/number columns (`NumberOfAccounts`, `DaysSinceChange`, `DaysSinceLogin`,
     `AccountsOnboarded`, `LatencyMs`, `ManagedPct`, …) → **Whole Number** / **Decimal**.
   - Boolean-like columns (`IsDefaultSafe`, `AutoManaged`, `HasCPM`, `IsEmpty`,
     `NamingCompliant`, `IsStale`, `RotationOverdue`, `IsComponent`, `IsLoggedOn`, …)
     → **leave as Text** (the DAX in this guide expects text).
4. **Close & Apply**.
5. Rename each query to match its filename without `.csv` (e.g. `FACT_Accounts`).

---

## Step 2: Relationships (Star Schema)

Open **Model view** (left sidebar, third icon). Create relationships by dragging the
*From* column onto the *To* column. Set cardinality/cross-filter in the dialog.

| # | From Table.Column | To Table.Column | Cardinality | Cross-filter |
|---|-------------------|-----------------|-------------|-------------|
| 1 | `FACT_Accounts.SafeName` | `DIM_Safe.SafeName` | Many → One | **Both** |
| 2 | `FACT_Accounts.PlatformID` | `DIM_Platform.PlatformID` | Many → One | **Both** |
| 3 | `FACT_Accounts.Region` | `DIM_Region.RegionKey` | Many → One | **Both** |
| 4 | `FACT_Accounts.CreatedDate` | `DIM_Date.DateKey` | Many → One | Single |
| 5 | `FACT_CPMFailed.PlatformID` | `DIM_Platform.PlatformID` | Many → One | Single |
| 6 | `FACT_CPMFailed.SafeName` | `DIM_Safe.SafeName` | Many → One | Single |
| 7 | `FACT_SafeMembers.SafeName` | `DIM_Safe.SafeName` | Many → One | Single |
| 8 | `FACT_AdminAudit.SafeName` | `DIM_Safe.SafeName` | Many → One | Single |
| 9 | `FACT_PlatformSummary.PlatformID` | `DIM_Platform.PlatformID` | Many → One | Single |
| 10 | `DIM_Safe.Region` | `DIM_Region.RegionKey` | Many → One | Single |

> [!NOTE]
> Do **not** relate `FACT_OnboardingTrend` or `FACT_VaultConnectivity` to anything.
> `DIM_Date.MonthKey` is not unique (one row per day), so the onboarding trend would be
> rejected — its visuals use `FACT_OnboardingTrend[Month]` directly instead.

---

## Step 3: DAX Measures

Create a home for measures: **Modeling → New table** → `_Measures = {BLANK()}`.
Then **Modeling → New measure** for each block below (paste one at a time).

### 3.1 Core Account Metrics
```dax
Total Accounts = COUNTROWS(FACT_Accounts)

Business Accounts =
CALCULATE(COUNTROWS(FACT_Accounts), FACT_Accounts[IsDefaultSafe] = "FALSE")

Compliant Accounts =
CALCULATE(COUNTROWS(FACT_Accounts),
    FACT_Accounts[ComplianceStatus] = "Compliant",
    FACT_Accounts[IsDefaultSafe] = "FALSE")

Non-Compliant Accounts =
CALCULATE(COUNTROWS(FACT_Accounts),
    FACT_Accounts[ComplianceStatus] = "Non-Compliant",
    FACT_Accounts[IsDefaultSafe] = "FALSE")

Pending Accounts =
CALCULATE(COUNTROWS(FACT_Accounts),
    FACT_Accounts[ComplianceStatus] = "Pending/Unknown",
    FACT_Accounts[IsDefaultSafe] = "FALSE")

Compliance Rate % = DIVIDE([Compliant Accounts], [Business Accounts], 0) * 100

APM Accounts =
CALCULATE(COUNTROWS(FACT_Accounts),
    FACT_Accounts[AutoManaged] = "TRUE",
    FACT_Accounts[IsDefaultSafe] = "FALSE")

MPM Accounts =
CALCULATE(COUNTROWS(FACT_Accounts),
    FACT_Accounts[AutoManaged] = "FALSE",
    FACT_Accounts[IsDefaultSafe] = "FALSE")

APM Rate % = DIVIDE([APM Accounts], [Business Accounts], 0) * 100
```

### 3.2 Rotation & Staleness
```dax
Rotation Overdue =
CALCULATE(COUNTROWS(FACT_Accounts),
    FACT_Accounts[RotationOverdue] = "TRUE",
    FACT_Accounts[IsDefaultSafe] = "FALSE")

Stale Accounts =
CALCULATE(COUNTROWS(FACT_Accounts),
    FACT_Accounts[IsStale] = "TRUE",
    FACT_Accounts[IsDefaultSafe] = "FALSE")

Never Verified =
CALCULATE(COUNTROWS(FACT_Accounts),
    FACT_Accounts[VerifyStatus] = "Never Verified",
    FACT_Accounts[IsDefaultSafe] = "FALSE")

Avg Password Age Days =
CALCULATE(
    AVERAGE(FACT_Accounts[DaysSinceChange]),
    FACT_Accounts[IsDefaultSafe] = "FALSE",
    NOT(ISBLANK(FACT_Accounts[DaysSinceChange])))
```

### 3.3 Safe Metrics (incl. SafeType)
```dax
Total Safes = COUNTROWS(DIM_Safe)

Business Safes =
CALCULATE(COUNTROWS(DIM_Safe), DIM_Safe[IsDefault] = "FALSE")

Empty Safes =
CALCULATE(COUNTROWS(DIM_Safe), DIM_Safe[IsEmpty] = "TRUE", DIM_Safe[IsDefault] = "FALSE")

Safes Without CPM =
CALCULATE(COUNTROWS(DIM_Safe), DIM_Safe[HasCPM] = "FALSE", DIM_Safe[IsDefault] = "FALSE")

Non-Standard Safe Names =
CALCULATE(COUNTROWS(DIM_Safe), DIM_Safe[NamingCompliant] = "FALSE", DIM_Safe[IsDefault] = "FALSE")

Recently Created Safes =
CALCULATE(COUNTROWS(DIM_Safe), DIM_Safe[RecentlyCreated] = "TRUE", DIM_Safe[IsDefault] = "FALSE")

Personal Admin Safes =
CALCULATE(COUNTROWS(DIM_Safe), DIM_Safe[SafeType] = "Personal Admin")

Shared Admin Safes =
CALCULATE(COUNTROWS(DIM_Safe), DIM_Safe[SafeType] = "Shared Admin")

Standard Business Safes =
CALCULATE(COUNTROWS(DIM_Safe), DIM_Safe[SafeType] = "Standard Business")
```

### 3.4 User Metrics
```dax
Total Users = COUNTROWS(FACT_Users)

Real Users =
CALCULATE(COUNTROWS(FACT_Users), FACT_Users[IsComponent] = "FALSE")

Active Users =
CALCULATE(COUNTROWS(FACT_Users),
    FACT_Users[IsComponent] = "FALSE",
    CONTAINSSTRING(FACT_Users[LoginStatus], "Active"))

Dormant Users =
CALCULATE(COUNTROWS(FACT_Users),
    FACT_Users[IsComponent] = "FALSE",
    CONTAINSSTRING(FACT_Users[LoginStatus], "Dormant"))

Never Logged In Users =
CALCULATE(COUNTROWS(FACT_Users),
    FACT_Users[IsComponent] = "FALSE",
    FACT_Users[LoginStatus] = "Never Logged In")

Active User Rate % = DIVIDE([Active Users], [Real Users], 0) * 100
```

### 3.5 CPM & Platform Metrics
```dax
CPM Failed Count = COUNTROWS(FACT_CPMFailed)

CPM Success Rate % =
VAR _managed = CALCULATE(COUNTROWS(FACT_Accounts),
    FACT_Accounts[AutoManaged] = "TRUE", FACT_Accounts[IsDefaultSafe] = "FALSE")
VAR _success = CALCULATE(COUNTROWS(FACT_Accounts),
    FACT_Accounts[AutoManaged] = "TRUE",
    FACT_Accounts[ComplianceStatus] = "Compliant",
    FACT_Accounts[IsDefaultSafe] = "FALSE")
RETURN DIVIDE(_success, _managed, 0) * 100
```

### 3.6 Vault Connectivity Metrics
```dax
Vault Components Total = COUNTROWS(FACT_VaultConnectivity)

Vault Components Connected =
CALCULATE(COUNTROWS(FACT_VaultConnectivity), FACT_VaultConnectivity[IsLoggedOn] = "TRUE")

Vault Connectivity % =
DIVIDE([Vault Components Connected], [Vault Components Total], 0) * 100
```

### 3.7 Onboarding Trend
```dax
Recently Onboarded Accounts =
CALCULATE(COUNTROWS(FACT_Accounts),
    FACT_Accounts[RecentlyOnboarded] = "TRUE",
    FACT_Accounts[IsDefaultSafe] = "FALSE")

Cumulative Accounts =
CALCULATE(
    SUM(FACT_OnboardingTrend[AccountsOnboarded]),
    FILTER(ALL(FACT_OnboardingTrend),
        FACT_OnboardingTrend[Month] <= MAX(FACT_OnboardingTrend[Month])))

Cumulative Safes =
CALCULATE(
    SUM(FACT_OnboardingTrend[SafesCreated]),
    FILTER(ALL(FACT_OnboardingTrend),
        FACT_OnboardingTrend[Month] <= MAX(FACT_OnboardingTrend[Month])))

Avg Monthly Onboarding = AVERAGE(FACT_OnboardingTrend[AccountsOnboarded])
```

### 3.8 Admin/Breakglass Audit (opt-in)
```dax
Admin Groups Compliance % =
VAR _total = COUNTROWS(FACT_AdminAudit)
VAR _full = CALCULATE(COUNTROWS(FACT_AdminAudit), FACT_AdminAudit[PermissionLevel] = "Full")
RETURN DIVIDE(_full, _total, 0) * 100

Admin Groups Missing =
CALCULATE(COUNTROWS(FACT_AdminAudit), FACT_AdminAudit[GroupExists] = "No")

Safe Member Assignments = COUNTROWS(FACT_SafeMembers)
```

### 3.9 Conditional-Formatting Color Helpers
> `TRUE()` here is the DAX **function** used by `SWITCH` — do not put it in quotes.
```dax
Compliance Color =
SWITCH(TRUE(),
    [Compliance Rate %] >= 90, "#10B981",
    [Compliance Rate %] >= 70, "#F59E0B",
    "#EF4444")

Connectivity Color =
SWITCH(TRUE(),
    [Vault Connectivity %] >= 95, "#10B981",
    [Vault Connectivity %] >= 80, "#F59E0B",
    "#EF4444")
```

---

## Step 4: Build the Dashboard Pages (detailed)

### 4.0 How to read these instructions (READ THIS FIRST)

Every Power BI visual has **field wells** — labelled boxes in the **Visualizations** pane
where you drag fields/measures. Which wells exist depends on the visual:

| Visual | Field wells (buckets) |
|--------|-----------------------|
| **Card** | *Fields* (one measure) |
| **Multi-row card** | *Fields* (several measures) |
| **Gauge** | *Value*, *Minimum*, *Maximum*, *Target* |
| **Clustered/Stacked _column_** (vertical bars) | *X-axis* = category, *Y-axis* = number, *Legend* = series, *Tooltips* |
| **Clustered/Stacked _bar_** (horizontal bars) | *Y-axis* = category, *X-axis* = number, *Legend* = series, *Tooltips* |
| **Line** | *X-axis* = category/date, *Y-axis* = number, *Legend* = series |
| **Line and stacked/clustered column (combo)** | *X-axis*, *Column y-axis*, *Line y-axis*, *Legend* |
| **Pie / Doughnut** | *Legend* = category, *Values* = number, *Detail*, *Tooltips* |
| **Treemap** | *Category*, *Details*, *Values*, *Tooltips* |
| **Matrix** | *Rows*, *Columns*, *Values* |
| **Table** | *Columns* |
| **Slicer** | *Field* |

**Two golden rules:**
1. A **category** (text you want to split/group by — e.g. `Region`, `SafeType`,
   `ComplianceStatus`) goes in *Axis / Legend / Rows / Columns / Category*.
2. A **number** (a measure like `[Business Accounts]`, or a numeric column you aggregate)
   goes in *Values / X-axis(bar) / Y-axis(column)*.

**"Count of X" vs a measure:** where a table below says *Values: count of AccountID*, drag the
column `FACT_Accounts[AccountID]` into the Values well, then click its dropdown → **Count**.
Where it says a measure like `[Business Accounts]`, drag that measure instead — no aggregation
needed. **Prefer the measures** where provided; they already exclude default/system objects.

**Visual-level filter:** to restrict one visual (not the whole page), select the visual, open
the **Filters** pane → **Filters on this visual** → drag the field there → pick the value
(e.g. `IsDefaultSafe` is `FALSE`). Type the value exactly (`FALSE`, case doesn't matter).

**Slicer = on-screen filter the user clicks.** Insert a **Slicer** visual, drop one field in it.

### 4.0.1 Recommended build order for every page
1. Set page background (**Format your report page → Canvas background →** color `#0F172A`, transparency 0%).
2. Add a **Text box** title top-left (16pt bold, white).
3. Add the **slicer row** across the top.
4. Add the **KPI card row** under the slicers.
5. Add the charts/tables in the body.
6. Apply formatting (Step 7 checklist).

### Color Palette
| Purpose | Hex |
|---|---|
| Primary (CyberArk Blue) | `#1E3A5F` |
| Accent (Teal) | `#0EA5E9` |
| Success | `#10B981` |
| Warning | `#F59E0B` |
| Error | `#EF4444` |
| Background | `#0F172A` |
| Card background | `#1E293B` |
| Text | `#F8FAFC` |
| Subtext | `#94A3B8` |

---

### 🏠 Page 1 — Executive Summary

**Slicers (top row):** insert 3 Slicer visuals.
- Slicer 1 → *Field*: `DIM_Region[RegionKey]`
- Slicer 2 → *Field*: `DIM_Safe[Tier]`
- Slicer 3 → *Field*: `DIM_Safe[SafeType]`

**KPI cards (row under slicers):** insert 6 **Card** visuals; drop one measure in each *Fields* well.
| Card | *Fields* (measure) | Conditional color (Format → Callout value → fx) |
|---|---|---|
| 1 | `[Business Accounts]` | — |
| 2 | `[Business Safes]` | — |
| 3 | `[Compliance Rate %]` | via `[Compliance Color]` |
| 4 | `[CPM Failed Count]` | green if 0, red if >0 |
| 5 | `[Active Users]` | — |
| 6 | `[Rotation Overdue]` | red if >0 |

**Body visuals:**

1. **Doughnut — Compliance split**
   - *Legend*: `FACT_Accounts[ComplianceStatus]`
   - *Values*: `FACT_Accounts[AccountID]` → set to **Count**
   - *Filter on this visual*: `FACT_Accounts[IsDefaultSafe]` is `FALSE`
   - Colours: Compliant=green, Non-Compliant=red, Pending/Unknown=amber.

2. **Stacked column — Accounts by Region, split by Tier**
   - *X-axis*: `DIM_Region[RegionKey]`
   - *Y-axis*: `FACT_Accounts[AccountID]` → **Count**  *(or the measure `[Business Accounts]`)*
   - *Legend*: `FACT_Accounts[Tier]`
   - *Filter on this visual*: `FACT_Accounts[IsDefaultSafe]` is `FALSE`

3. **Doughnut — Safe Type split** *(new)*
   - *Legend*: `DIM_Safe[SafeType]`
   - *Values*: `DIM_Safe[SafeName]` → **Count**
   - *Filter*: `DIM_Safe[IsDefault]` is `FALSE`

4. **Doughnut — APM vs MPM**
   - *Legend*: `FACT_Accounts[MgmtTechnique]`
   - *Values*: `FACT_Accounts[AccountID]` → **Count**
   - *Filter*: `FACT_Accounts[IsDefaultSafe]` is `FALSE`

5. **Bar (horizontal) — Top 10 safes by account count**
   - *Y-axis*: `DIM_Safe[SafeName]`
   - *X-axis*: `DIM_Safe[NumberOfAccounts]` → **Sum**
   - *Filter*: `DIM_Safe[IsDefault]` is `FALSE`; and **Top N** filter → Top 10 by
     `NumberOfAccounts` (Filters pane → SafeName → Filter type = Top N).

6. **Multi-row card — watch list**
   - *Fields* (add all four): `[Rotation Overdue]`, `[Stale Accounts]`,
     `[Non-Standard Safe Names]`, `[Empty Safes]`.

---

### 📊 Page 2 — Account Inventory & Compliance

**Slicers:** `DIM_Region[RegionKey]`, `DIM_Safe[Tier]`, `DIM_Safe[SafeType]`,
`DIM_Platform[PlatformID]`, `FACT_Accounts[OSCategory]`, `FACT_Accounts[ComplianceStatus]`.

**KPI cards:** `[Business Accounts]`, `[Compliance Rate %]` (use `[Compliance Color]`),
`[APM Rate %]`, `[Rotation Overdue]` (red if >0), `[Never Verified]` (red if >0).

**Body visuals:**

1. **Matrix — Region × Tier**
   - *Rows*: `FACT_Accounts[Region]`
   - *Columns*: `FACT_Accounts[Tier]`
   - *Values*: `FACT_Accounts[AccountID]` → **Count**
   - *Filter*: `IsDefaultSafe` is `FALSE`
   - Format → **Cell elements → Background color → On** (heat-map effect).

2. **Stacked column — Compliance by OS**
   - *X-axis*: `FACT_Accounts[OSCategory]`
   - *Y-axis*: `FACT_Accounts[AccountID]` → **Count**
   - *Legend*: `FACT_Accounts[ComplianceStatus]`
   - *Filter*: `IsDefaultSafe` is `FALSE`

3. **Clustered column — Verify status by Region**
   - *X-axis*: `FACT_Accounts[Region]`
   - *Y-axis*: `FACT_Accounts[AccountID]` → **Count**
   - *Legend*: `FACT_Accounts[VerifyStatus]`
   - *Filter*: `IsDefaultSafe` is `FALSE`

4. **Treemap — Accounts by Platform**
   - *Category*: `FACT_Accounts[PlatformID]`
   - *Values*: `FACT_Accounts[AccountID]` → **Count**
   - *Details* (optional): `FACT_Accounts[OSCategory]`

5. **Table — account detail**
   - *Columns* (in order): `Name`, `UserName`, `Address`, `SafeName`, `SafeType`,
     `PlatformID`, `Region`, `Tier`, `ComplianceStatus`, `DaysSinceChange`,
     `RotationOverdue`, `IsStale`, `VerifyStatus` — all from `FACT_Accounts`.
   - *Filter*: `IsDefaultSafe` is `FALSE`.

---

### 📈 Page 3 — Onboarding Trend & Growth

**Slicers:** `DIM_Region[RegionKey]`, `DIM_Safe[Tier]`.

**KPI cards:** `[Recently Onboarded Accounts]`, `[Recently Created Safes]`,
`[Avg Monthly Onboarding]` (format 1 decimal), `[Business Accounts]`.

**Body visuals:**

1. **Line and clustered column (combo) — onboarding over time**
   - *X-axis*: `FACT_OnboardingTrend[Month]`
   - *Column y-axis*: `FACT_OnboardingTrend[AccountsOnboarded]` → **Sum**
   - *Line y-axis*: `[Cumulative Accounts]`

2. **Clustered column — safes created per month**
   - *X-axis*: `FACT_OnboardingTrend[Month]`
   - *Y-axis*: `FACT_OnboardingTrend[SafesCreated]` → **Sum**

3. **Line — cumulative safes**
   - *X-axis*: `FACT_OnboardingTrend[Month]`
   - *Y-axis*: `[Cumulative Safes]`

4. **Stacked column — accounts onboarded per month by Region**
   - *X-axis*: `FACT_Accounts[CreatedMonth]`
   - *Y-axis*: `FACT_Accounts[AccountID]` → **Count**
   - *Legend*: `FACT_Accounts[Region]`
   - *Filters*: `IsDefaultSafe` is `FALSE`; `CreatedMonth` **is not blank**.

---

### 🔐 Page 4 — Safe Inventory & Naming Compliance

**Slicers:** `DIM_Region[RegionKey]`, `DIM_Safe[Tier]`, `DIM_Safe[SafeType]`,
`DIM_Safe[HasCPM]`, `DIM_Safe[IsEmpty]`.

**KPI cards:** `[Business Safes]`, `[Empty Safes]` (amber if >0),
`[Safes Without CPM]` (red if >0), `[Non-Standard Safe Names]` (red if >0),
`[Recently Created Safes]`.

**Body visuals:**

1. **Pie — safes by Region**
   - *Legend*: `DIM_Safe[Region]`; *Values*: `DIM_Safe[SafeName]` → **Count**;
     *Filter*: `IsDefault` is `FALSE`.

2. **Stacked column — SafeType vs naming compliance** *(new)*
   - *X-axis*: `DIM_Safe[SafeType]`
   - *Y-axis*: `DIM_Safe[SafeName]` → **Count**
   - *Legend*: `DIM_Safe[NamingCompliant]` (TRUE=green, FALSE=red)
   - *Filter*: `IsDefault` is `FALSE`.

3. **Bar (horizontal) — Top 10 safes by account count**
   - *Y-axis*: `DIM_Safe[SafeName]`; *X-axis*: `DIM_Safe[NumberOfAccounts]` → **Sum**;
     *Filter*: `IsDefault` is `FALSE` + **Top N** 10 by `NumberOfAccounts`.

4. **Doughnut — CPM coverage**
   - *Legend*: `DIM_Safe[HasCPM]`; *Values*: `DIM_Safe[SafeName]` → **Count**;
     *Filter*: `IsDefault` is `FALSE`.

5. **Table — safe detail**
   - *Columns*: `SafeName`, `SafeType`, `Region`, `Tier`, `ManagingCPM`,
     `NumberOfAccounts`, `HasCPM`, `IsEmpty`, `NamingCompliant`, `CreatedDate` (from `DIM_Safe`).
   - *Filter*: `IsDefault` is `FALSE`.
   - Format → **Cell elements** on the `NamingCompliant` column → Background color rule:
     value `FALSE` → red.

---

### 👥 Page 5 — Users, Groups & Access Governance

**Slicers:** `FACT_Users[LoginStatus]`, `FACT_Users[UserType]`.

**KPI cards:** `[Real Users]`, `[Active Users]` (green), `[Dormant Users]` (amber),
`[Never Logged In Users]` (red), and a Card with *Fields* = `FACT_Groups[GroupName]` → **Count**
(rename its title to "Groups").

**Body visuals:**

1. **Doughnut — login-status split**
   - *Legend*: `FACT_Users[LoginStatus]`; *Values*: `FACT_Users[UserID]` → **Count**;
     *Filter*: `IsComponent` is `FALSE`. Colours: Active=green, Dormant=amber, Never=red.

2. **Bar (horizontal) — vault authorizations**
   - *Y-axis*: `FACT_Users[VaultAuth]`; *X-axis*: `FACT_Users[UserID]` → **Count**;
     *Filter*: `IsComponent` is `FALSE`.

3. **Pie — groups default vs custom**
   - *Legend*: `FACT_Groups[IsDefault]`; *Values*: `FACT_Groups[GroupName]` → **Count**.

4. **Table — user detail**
   - *Columns*: `UserName`, `LoginStatus`, `DaysSinceLogin`, `Source`, `Enabled`,
     `Suspended` (from `FACT_Users`); *Filter*: `IsComponent` is `FALSE`;
     sort by `DaysSinceLogin` descending (click the column header).

---

### 🖥️ Page 6 — Vault Connectivity

Only `FACT_VaultConnectivity` feeds this page (CPM / PSM / AIM logon status).

**Slicer:** `FACT_VaultConnectivity[ComponentType]`.

**KPI cards:** `[Vault Components Total]`, `[Vault Components Connected]`,
`[Vault Connectivity %]` (use `[Connectivity Color]`).

**Body visuals:**

1. **Clustered column — connected vs not, by component type**
   - *X-axis*: `FACT_VaultConnectivity[ComponentType]`
   - *Y-axis*: `FACT_VaultConnectivity[InstanceIP]` → **Count**
   - *Legend*: `FACT_VaultConnectivity[IsLoggedOn]` (TRUE=green, FALSE=red)

2. **Table — connectivity detail**
   - *Columns*: `ComponentType`, `InstanceIP`, `VaultUserName`, `ComponentVersion`,
     `IsLoggedOn`, `LastLogonDate`.
   - Format → **Cell elements** on `IsLoggedOn` → Background color: value `FALSE` → red.

---

### 🔍 Page 7 — CPM Operations & Failures

**Slicers:** `DIM_Platform[PlatformID]`, `DIM_Region[RegionKey]`, `DIM_Safe[Tier]`.

**KPI cards:** `[CPM Failed Count]` (red if >0), `[CPM Success Rate %]`,
`[APM Accounts]`, and a Card with `FACT_PlatformSummary[PlatformID]` → **Count** (title "Platforms Tracked").

**Body visuals:**

1. **Bar (horizontal) — top failing platforms**
   - *Y-axis*: `FACT_PlatformSummary[PlatformID]`
   - *X-axis*: `FACT_PlatformSummary[FailedCount]` → **Sum**
   - *Filter*: **Top N** 10 by `FailedCount`.

2. **Clustered column — managed vs unmanaged by platform**
   - *X-axis*: `FACT_PlatformSummary[PlatformID]`
   - *Y-axis*: add both `FACT_PlatformSummary[ManagedCount]` and
     `FACT_PlatformSummary[UnmanagedCount]` → **Sum**
   - *Filter*: **Top N** 10 by `TotalAccounts`.

3. **Stacked column — CPM failures by Region**
   - *X-axis*: `FACT_CPMFailed[Region]`
   - *Y-axis*: `FACT_CPMFailed[AccountID]` → **Count**
   - *Legend*: `FACT_CPMFailed[CPMStatus]`

4. **Doughnut — CPM failures by Tier**
   - *Legend*: `FACT_CPMFailed[Tier]`; *Values*: `FACT_CPMFailed[AccountID]` → **Count**.

5. **Table — failed-account detail**
   - *Columns*: `AccountName`, `Address`, `PlatformID`, `SafeName`, `Region`, `Tier`,
     `CPMStatus`, `ManualReason`, `LastModified`, `LastVerified` (from `FACT_CPMFailed`).

---

### 🧾 Page 8 *(opt-in)* — Safe Membership & Admin Audit

Only has data when the script ran with `$CollectSafeMembers = $true`
(uses `FACT_SafeMembers` + `FACT_AdminAudit`).

**Slicers:** `DIM_Region[RegionKey]`, `DIM_Safe[Tier]`, `DIM_Safe[SafeType]`,
`FACT_SafeMembers[MemberType]`.

**KPI cards:** `[Safe Member Assignments]`, `[Admin Groups Compliance %]`,
`[Admin Groups Missing]` (red if >0).

**Body visuals:**

1. **Stacked column — members by Region, split by type**
   - *X-axis*: `FACT_SafeMembers[Region]`
   - *Y-axis*: `FACT_SafeMembers[MemberName]` → **Count**
   - *Legend*: `FACT_SafeMembers[MemberType]` (User / Group)

2. **Doughnut — member category split**
   - *Legend*: `FACT_SafeMembers[MemberCategory]`; *Values*: `FACT_SafeMembers[MemberName]` → **Count**.

3. **Table — admin/breakglass group audit**
   - *Columns*: `SafeName`, `GroupName`, `GroupExists`, `PermissionLevel`,
     `MissingPermissions` (from `FACT_AdminAudit`).
   - Format → **Cell elements** on `PermissionLevel`: `Partial` → amber; and on
     `GroupExists`: `No` → red.

4. **Table — member detail**
   - *Columns*: `SafeName`, `MemberName`, `MemberType`, `IsSafeManager`,
     `MemberCategory`, `Permissions`, `MembershipExpirationDate` (from `FACT_SafeMembers`).
   - *Filter*: `IsDefaultSafe` is `FALSE`.

---

## Step 5: Global Slicer Sync

**View → Sync slicers.** Select each slicer and tick the pages it should apply to:
- `DIM_Region[RegionKey]` → Pages 1, 2, 3, 4, 7, 8
- `DIM_Safe[Tier]` → Pages 1, 2, 3, 4, 7, 8
- `DIM_Safe[SafeType]` → Pages 1, 2, 4, 8
- `DIM_Platform[PlatformID]` → Pages 2, 7

---

## Step 6: Page Navigation

**Insert → Buttons → Navigator → Page navigator.** Style it as an icon sidebar
(dark background, active page in accent `#0EA5E9`):
🏠 Executive · 📊 Accounts · 📈 Onboarding · 🔐 Safes · 👥 Users · 🖥️ Vault · 🔍 CPM · 🧾 Membership.

---

## Step 7: Formatting Checklist

- [ ] Page background `#0F172A`
- [ ] Card backgrounds `#1E293B`, 8px rounded
- [ ] Card titles 10pt `#94A3B8`; values 24–28pt `#F8FAFC` bold
- [ ] Charts: transparent background; axis labels `#94A3B8`; data labels `#F8FAFC` on; gridlines `#334155`
- [ ] Tables: header `#1E3A5F`/white; alternating rows `#1E293B` / `#0F172A`
- [ ] Slicers: dropdown style, dark background, white text
- [ ] Last-refresh card: new Card, *Fields* = `MAX(FACT_Accounts[ExtractDate])`
- [ ] Page title text box 16pt bold `#F8FAFC` top-left

---

## Quick Reference: Which Table Feeds Which Page

| Page | Primary Tables | Slicer Tables |
|------|---------------|---------------|
| 1 - Executive | `FACT_Accounts`, `DIM_Safe`, `DIM_Region` | `DIM_Region`, `DIM_Safe` |
| 2 - Accounts | `FACT_Accounts`, `DIM_Platform` | `DIM_Region`, `DIM_Safe`, `DIM_Platform` |
| 3 - Onboarding | `FACT_OnboardingTrend`, `FACT_Accounts` | `DIM_Region`, `DIM_Safe` |
| 4 - Safes | `DIM_Safe` | `DIM_Region`, `DIM_Safe` |
| 5 - Users | `FACT_Users`, `FACT_Groups` | `FACT_Users` |
| 6 - Vault Connectivity | `FACT_VaultConnectivity` | `FACT_VaultConnectivity` |
| 7 - CPM | `FACT_CPMFailed`, `FACT_PlatformSummary` | `DIM_Platform`, `DIM_Region` |
| 8 - Membership *(opt-in)* | `FACT_SafeMembers`, `FACT_AdminAudit` | `DIM_Region`, `DIM_Safe` |
