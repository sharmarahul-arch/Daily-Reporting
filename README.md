# Amazon Daily Reporting Tool

Automatically pulls yesterday's data from Amazon Seller Central + Amazon Ads for every account and uploads it to the account's Google Sheet — no manual work needed.

**What it collects per account:**
| Tab | Source | Rows |
|---|---|---|
| Sales | Seller Central → Business Reports | One row per ASIN |
| Ad Campaigns | Amazon Ads → Campaign Manager | One row per campaign |
| Advertised Products | Amazon Ads → Products tab | One row per product |

---

## Quick start (first-time setup)

### 1. Clone / copy the project folder
Put it anywhere on your machine, e.g. `~/Documents/Daily Reporting`.

### 2. Install dependencies
```bash
cd "Daily Reporting"
bash setup.sh
```
This creates a virtual environment, installs all Python packages, and installs the Chromium browser.

### 3. Fill in your credentials — `config/credentials.json`

Open `config/credentials.json` and replace the values:

```json
{
  "amazon_email": "your-amazon-login@email.com",
  "amazon_password": "YourPassword",
  "sheets_service_account_path": "/absolute/path/to/service-account.json"
}
```

- **amazon_email / amazon_password** — the Amazon account that has access to all the Seller Central accounts you want to report on.
- **sheets_service_account_path** — path to the Google service account JSON key file (see step 4).

> ⚠️ Never commit `credentials.json` to Git — it contains your password.

### 4. Set up Google Sheets access (one-time)

1. Go to [Google Cloud Console](https://console.cloud.google.com/) → your project → **IAM & Admin → Service Accounts**.
2. Create a service account, give it no roles, then create a JSON key and download it.
3. Put the downloaded `.json` file somewhere safe and paste its **full path** into `credentials.json` (step 3).
4. For **each** Google Sheet the tool will write to, share the sheet with the service account email (it looks like `name@project.iam.gserviceaccount.com`) and give it **Editor** access.

### 5. Add your accounts — `config/accounts.json`

This is the main file you edit to add, remove, or pause accounts.

```json
[
  {
    "name": "Brand Display Name",
    "sc_account_name": "Exact name shown in SC account switcher",
    "sc_parent": null,
    "ads_account_name": "Exact name shown in Amazon Ads entity switcher",
    "marketplace": "IN",
    "google_sheet_id": "SHEET_ID_FROM_URL",
    "active": true
  }
]
```

**Field guide:**

| Field | What to put | Example |
|---|---|---|
| `name` | Any label you want | `"HEM Agarbatti"` |
| `sc_account_name` | Exact text in SC account switcher (or merchant ID) | `"A20WE5N42SW9UE"` |
| `sc_parent` | Parent SPN name if the account is nested; `null` otherwise | `"UPRIVER SPN"` or `null` |
| `ads_account_name` | Exact text in Amazon Ads entity switcher | `"HEM Agarbatti"` |
| `marketplace` | `"IN"` for India, `"US"` for United States | `"IN"` |
| `google_sheet_id` | The long ID in the sheet URL: `docs.google.com/spreadsheets/d/**THIS_PART**/edit` | `"10eSn2MKv7AB..."` |
| `active` | `true` to include in daily run, `false` to skip | `true` |

### 6. Test your Sheets connection
```bash
source venv/bin/activate
python main.py --test-sheets YOUR_SHEET_ID
```

### 7. Run manually
```bash
source venv/bin/activate
python main.py                         # all active accounts
python main.py --account "HEM Agarbatti"   # one account only
```

On first run the browser will open and ask for OTP/2FA. Complete it in the window — the session is then saved and reused automatically for future runs.

### 8. Schedule the daily run (optional)
```bash
python scheduler.py               # runs every day at 7:00 AM
python scheduler.py --time 06:30  # custom time
```

---

## Project structure

```
Daily Reporting/
├── config/
│   ├── accounts.json        ← ADD YOUR ACCOUNTS HERE
│   └── credentials.json     ← YOUR LOGIN & SERVICE ACCOUNT PATH (keep private)
├── src/
│   ├── config.py            — loads settings
│   ├── auth.py              — Amazon login & account switching
│   ├── seller_reports.py    — Seller Central report download
│   ├── ads_reports.py       — Amazon Ads report download
│   ├── sheets.py            — Google Sheets upload
│   └── utils.py             — logging, date helpers
├── logs/                    ← daily run logs (auto-created)
├── downloads/               ← temp CSVs (auto-cleaned after each run)
├── sessions/                ← saved login cookies (auto-managed)
├── main.py                  — entry point
├── scheduler.py             — daily auto-runner
├── requirements.txt
└── setup.sh                 — first-time install script
```

---

## Common issues

| Problem | Fix |
|---|---|
| OTP prompt on every run | Complete the OTP once — the session saves automatically. If it keeps asking, run `python main.py --clear-sessions` to reset. |
| "Account not found" in SC switcher | Check `sc_account_name` exactly matches the text shown in the SC account switcher dropdown. |
| "Entity not found" in Ads | Check `ads_account_name` exactly matches the entity name in Amazon Ads (visible when you click the account switcher in the top-left). |
| Wrong data date | The tool always pulls **yesterday's** data. Run it any time after midnight IST. |
| Google Sheets permission error | Make sure the sheet is shared with your service account email (Editor access). |
| Chrome already open error | Close all Chrome windows before running — Playwright needs exclusive access to the Chrome profile. |
