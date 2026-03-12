# open-source
Copies of internal scripts used at KCRHA that other communities may find helpful

---

## Scripts

### [All_Program_Enrollments.ipynb](All_Program_Enrollments.ipynb)

**Purpose:** Produces the foundational enrollment-level analytic table for the Data Hub. Most other analytic tables downstream link to the output of this script.

**What it does:**
- Pulls all HMIS enrollment records (2017–present) from Looker across all 15 project type codes using concurrent requests with exponential backoff retry logic
- Standardizes and renames columns to organizational conventions
- Cleans head-of-household designations — handles households with missing or duplicate HOH records by assigning the oldest adult member as HOH
- Derives `HouseholdType` (with children, without children, only children) and `HouseholdCategory` (Family with Children, Youth and Young Adults, Individual Adults) from age data
- Outputs an enrollment-level Parquet file to Azure Blob Storage

**Output:** One row per enrollment. Serves as the base join table for the rest of the Data Hub.

---

### [Event_Systemwide.ipynb](Event_Systemwide.ipynb)

**Purpose:** Transforms raw HMIS data from multiple sources into a unified, chronological event stream that tracks client movement into and out of homelessness. This is the foundation that feeds `Episode_Systemwide`.

**What it does:**
- Pulls enrollment data, Current Living Situation (CLS) records, and services data from Looker
- Builds event records from seven distinct sources:
  - CLS assessments
  - Homeless program enrollment start dates (filtered by prior living situation and fleeing status)
  - Move-in dates from Permanent Housing programs (Housed events)
  - Homelessness Prevention program enrollments (Housed events)
  - Service attendance records (homeless and housed service types)
  - Ongoing enrollment midpoint events (to ensure multi-month enrollments span their full episode)
  - Deceased exits
- Assigns each event a `ClientStatus` (Homeless / Housed / Deceased), `ClientShelterStatus`, and `EventType`
- Deduplicates events per person per date using a priority hierarchy
- Generates a unique `EventID` per event via MD5 hash of PersonalID + EventDate + EnrollmentID
- Outputs a Parquet file to Azure Blob Storage

**Output:** One row per event per person per date. Long-form event stream across the full system.

---

### [Episode_Systemwide.ipynb](Episode_Systemwide.ipynb)

**Purpose:** Converts the event stream from `Event_Systemwide` into discrete episodes of homelessness — continuous periods during which a person is classified as homeless — and scaffolds those episodes across calendar months to support time-period-based reporting.

**What it does:**
- Reads `event_systemwide` and `all_program_enrollments` Parquet files from Azure Blob Storage
- Pulls CE assessment dates and client birth dates from Looker
- Segments each person's event history into episodes based on:
  - Status changes (e.g., Homeless → Housed)
  - Inactivity gaps exceeding a configurable threshold (default: 30 days)
- Classifies episode **inflow type** (Newly Homeless, Return from Housed, Return from Inactive) and **outflow type** (Active, Permanently Housed, Deceased, Inactive)
- Scaffolds each episode into one row per calendar month, enabling flexible monthly/quarterly/yearly reporting
- Flags whether each episode overlapped with a Coordinated Entry enrollment and whether a CE assessment was present
- Calculates `AgedOutOfYYA` — the date a person turned 25 if that birthday falls within the episode
- Joins in client names and unique identifiers
- Outputs a Parquet file to Azure Blob Storage

**Output:** One row per person per episode per calendar month. Designed to feed by-name lists, performance dashboards, and system flow metrics.

> **Note:** The output of this script includes PII (client first/last names and PersonalIDs). Ensure appropriate access controls are applied to the destination storage container.

---

## Security Notes

- All credentials and environment-specific values are templated as `[[PLACEHOLDER]]` — replace with your organization's values before use.
- Production deployments use Azure Key Vault for secrets (`TokenLibrary.getSecretWithLS`). The development path uses `DefaultAzureCredential` (requires `az login`).
- Output Parquet files are written with `overwrite=True` — each pipeline run replaces the prior output. Verify destination paths before running in production.
- All Looker queries use `limit: -1` (no row cap). On very large HMIS datasets, monitor memory usage.
