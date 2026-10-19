# Crowdstrike USB Device Control Usage

> A reporting toolkit that audits CrowdStrike **Device Control USB exceptions** (the allowlist) against **real-world device usage**, surfacing which exceptions are actually in use and which are dormant, so stale allowlist entries can be reviewed and revoked.

<p align="center">
  <img src="Files/usb-audit-report.png" alt="Example USB audit report: each Device Control exception with its Active/Dormant verdict, last-seen date, connected machines, and match method.">
</p>

<table width="100%">
<thead>
<tr><th align="left">Before<div><img width="300" height="1" alt=""></div></th><th align="left">Now<div><img width="300" height="1" alt=""></div></th><th align="left">What it buys<div><img width="400" height="1" alt=""></div></th></tr>
</thead>
<tbody>
<tr><td>~7 days of usage history — too short a window to know if an exception is still used</td><td>a 6-month rolling history built weekly, cross-referenced against the live allowlist</td><td>stale exceptions found and ready for review instead of sitting open indefinitely</td></tr>
</tbody>
</table>

---

## Overview

CrowdStrike Device Control lets you create **exceptions** to allow specific USB devices past your block policy. Over time these allowlists grow, and nobody knows which exceptions are still being used. Every unused exception is an open door that no longer needs to be open.

This toolkit correlates two data sources from the Falcon platform, the **allowlist** (what is permitted) and the **real usage** (what was actually plugged in), and produces a periodic audit that flags each exception as **active** or **dormant**, turning USB least-privilege hygiene from a guess into evidence.

## How It Works

CrowdStrike retains only **~7 days** of device usage history, too short a window to tell whether an exception is genuinely unused. The toolkit works around this with a two-stage pipeline:

- **Stage 1. Collect.** A weekly job pulls the latest 7-day window of USB activity and appends it to an accumulated dataset (retained up to **6 months**), building the long-term history Falcon itself doesn't keep. A second job snapshots the current exception allowlist.
- **Stage 2. Audit.** The audit cross-references the allowlist against whatever usage history has accumulated and flags each exception as active or dormant. It can be run at any time; the report records how many days of history it was based on (`Usage History (Days)`) so results can be interpreted accordingly.

> ⏳ **Run the weekly collector consistently.** Because Falcon keeps only ~7 days, any missed week becomes a permanent gap in the history, and the audit is only as good as the history behind it.

---

<details>
<summary><strong>Requirements</strong> (click to expand)</summary>

- **CrowdStrike Falcon** with **Device Control** and **NGSIEM** enabled
- A Falcon **API client** with Device Control and NGSIEM read/write scopes

</details>

<details>
<summary><strong>Setup</strong> (click to expand)</summary>

### 1. Environment and dependencies

```bash
# Virtual environment
python -m venv .venv
.venv\Scripts\Activate.ps1      # Windows (PowerShell)
source .venv/bin/activate        # Linux / macOS

# Install dependencies
pip install -r Scripts/requirements.txt
```

<table width="100%">
<thead>
<tr><th align="left">Dependency<div><img width="300" height="1" alt=""></div></th><th align="left">Purpose<div><img width="900" height="1" alt=""></div></th></tr>
</thead>
<tbody>
<tr><td><code>crowdstrike-falconpy</code></td><td>Official CrowdStrike Falcon SDK (Device Control + NGSIEM APIs)</td></tr>
<tr><td><code>pandas</code></td><td>Merges the usage history, cross-references the allowlist, and writes the CSVs</td></tr>
<tr><td><code>python-dotenv</code></td><td>Loads API credentials from <code>.env</code></td></tr>
</tbody>
</table>

### 2. API credentials

Create `Scripts/Falcon/.env` with a Falcon API client that has **Device Control** and **NGSIEM** read scopes:

```env
CLIENT_ID="your-falcon-client-id"
SECRET_ID="your-falcon-client-secret"
```

> ⚠️ **Never commit `.env`.** Add it to `.gitignore`. If credentials were ever committed, revoke and rotate them in the Falcon console.

</details>

<details>
<summary><strong>Usage</strong> (click to expand)</summary>

The pipeline has three scripts. Run them in order:

### 1. `Extract_Weekly_Usage.py` *(schedule weekly)*

```bash
python Scripts/Falcon/Extract_Weekly_Usage.py
```
→ `Datas/Entry/Device-Control-Usage_<range>.csv`

### 2. `Extract_USB_Devices.py`

```bash
python Scripts/Falcon/Extract_USB_Devices.py
```
→ `Datas/Entry/Device-Control-USB-Exceptions.csv`

> ⚠️ Collects only exceptions defined **within policies**. **CID-level (Custom) exceptions** are not retrieved and must be accounted for separately if your environment uses them.

### 3. `Generate_USB-Audit-Usage.py`

```bash
python Scripts/Generate_USB-Audit-Usage.py
```
→ `Datas/<YYYY-MM>/Device_USB-Audit-Usage.csv`

> 🔧 The audit can be run at any time; there is no minimum history requirement. The report includes a `Usage History (Days)` column showing how many days of accumulated usage the result is based on.
>
> ✅ The audit matches all three exception types: **`COMBINED_ID`** (by `Device Combined Id`), **`VID_PID_SERIAL`** (by vendor + product + serial), and **`VID_PID`** (by vendor + product). `VID_PID` matches are broader by nature: any device sharing that vendor/product counts as usage.

**Suggested cadence:** schedule step 2 **weekly**; refresh step 1 whenever you audit; run step 3 **monthly**. The more history accumulated, the more reliable the active/dormant verdict.

</details>

<details>
<summary><strong>Audit Report Fields</strong> (click to expand)</summary>

The generated `Device_USB-Audit-Usage.csv` includes the full exception definition plus the usage-derived columns:

<table width="100%">
<thead>
<tr><th align="left">Field<div><img width="450" height="1" alt=""></div></th><th align="left">Description<div><img width="750" height="1" alt=""></div></th></tr>
</thead>
<tbody>
<tr><td><code>Usage History (Days)</code></td><td>Number of days of accumulated usage history the audit is based on (span between the oldest and newest usage record)</td></tr>
<tr><td><code>Is Active</code></td><td><code>True</code> if the device was seen with <code>Full access</code> usage</td></tr>
<tr><td><code>Last Seen</code></td><td>Most recent usage timestamp for the device</td></tr>
<tr><td><code>Last Connected Machines</code></td><td>Hostnames that connected the device</td></tr>
<tr><td><code>Connection Type</code></td><td>USB / Storage spaces / PCIe</td></tr>
<tr><td><code>Match Method</code></td><td>How the exception is matched against usage: <code>COMBINED_ID</code>, <code>VID_PID_SERIAL</code>, or <code>VID_PID</code></td></tr>
<tr><td><code>Device Combined Id</code> / <code>Device Vendor Id</code> / <code>Device Product Id</code> / <code>Device Serial Id</code></td><td>Device identifiers; which ones are used for matching depends on <code>Match Method</code></td></tr>
<tr><td><code>Exception Id</code> / <code>Exception Action</code></td><td>The Device Control exception and its action</td></tr>
<tr><td><code>Policy Name</code> / <code>Policy Platform</code> / <code>Policy Enabled</code></td><td>Owning Device Control policy context</td></tr>
<tr><td><code>USB Enforcement Mode</code> / <code>USB Device Class</code></td><td>Policy enforcement details</td></tr>
<tr><td><code>Created At</code> / <code>Modified At</code></td><td>Exception lifecycle timestamps</td></tr>
<tr><td><code>Reference Date</code> / <code>Reference Month</code> / <code>Reference Year</code></td><td>When the audit was generated</td></tr>
</tbody>
</table>

</details>

<details>
<summary><strong>File Structure</strong> (click to expand)</summary>

```
Crowdstrike-USB-Device-Control-Usage/
├── Scripts/
│   ├── Falcon/
│   │   ├── Extract_USB_Devices.py     # Pulls Device Control USB exceptions (API)
│   │   ├── Extract_Weekly_Usage.py    # Pulls weekly USB usage events (NGSIEM)
│   │   └── .env                       # Falcon API credentials (not committed)
│   ├── Generate_USB-Audit-Usage.py    # Cross-references exceptions vs usage
│   └── requirements.txt
├── Datas/
│   ├── Entry/                         # Raw collected CSVs (exceptions + usage)
│   └── <YYYY-MM>/                     # Generated monthly audit reports
└── Files/
    └── usb-audit-report.png           # Example: generated USB audit report
```

</details>

<details>
<summary><strong>Roadmap / Next Steps</strong> (click to expand)</summary>

- **Automated cleanup.** Once the environment is well understood and the audit is trusted, a script that automatically **removes dormant exceptions** via the Device Control API, closing the loop from *detection* to *remediation*.
- **Include CID-level exceptions** in the allowlist extract for full coverage.

</details>

---

## License

This toolkit is released under the MIT License: use, adapt and redistribute it freely. Full terms in [LICENCE.md](LICENCE.md).
