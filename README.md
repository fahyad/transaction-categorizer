# Transaction Categorizer

A mobile-friendly web app for categorizing Scotiabank transactions and exporting them to a budget spreadsheet. Hosted on GitHub Pages, it connects to a Google Apps Script web app that parses Scotiabank InfoAlert emails from Gmail, returns uncategorized transactions, and writes categories back — all in one call.

**Live app:** [fahyad.github.io/transaction-categorizer](https://fahyad.github.io/transaction-categorizer/)

---

## How It Works

### Architecture

```
Scotiabank InfoAlert emails
        |
        v
  Gmail (forwarded)
        |
        v
  Google Apps Script (backend)
    - Triggered by the app via ?action=parseAndFetch
    - Parses unread emails with regex
    - Logs transactions to Google Sheet (persistent record)
    - Returns uncategorized rows as JSON
        |
        v
  Transaction Categorizer (this app)
    - Triggers email parsing + fetches results in one call
    - Multi-select confirm for known merchants
    - One-at-a-time categorization for new merchants
    - POSTs category updates back (with retry queue)
    - Copies formatted output to clipboard for Excel
```

### Data Flow

1. **Scotiabank** sends transaction alert emails (InfoAlerts) to your email.
2. Emails are forwarded to **Gmail**.
3. When you tap **Fetch Transactions**, the app calls the Apps Script `doGet` endpoint with `?action=parseAndFetch`.
4. The Apps Script **parses any new unread emails** from Gmail, writes them to the Google Sheet, then returns all uncategorized transactions as JSON.
5. Known merchants are auto-suggested — you confirm or reject them in a **multi-select batch screen**.
6. Unknown merchants are categorized one at a time with a grid of category buttons.
7. Each category assignment is **POSTed back** to the Apps Script in real-time (with automatic retry on failure).
8. A **Copy to Clipboard** button formats categorized transactions as tab-separated text for pasting into your Excel budget workbook.

---

## Features

- **One-tap fetch + parse** — triggers email parsing and returns uncategorized transactions in a single call (no manual script runs needed).
- **Sync status indicator** — shows real-time sync status for category updates, with automatic retry (up to 3x) and a manual retry button for failures.
- **Richer transaction cards** — displays time (ET), transaction type badge (authorization, purchase, etc.), and account info alongside payee and amount.
- **Multi-select batch confirm** — known merchants are shown as a scrollable checklist; select/deselect individually or all at once, then confirm or reject in bulk.
- **Refresh for new transactions** — check for new uncategorized transactions from the summary screen without restarting the session.
- **Pending count + filters** — auto-peeks uncategorized count on load, with account filter and date sort (newest/oldest first) before fetching.
- **CSV upload** — secondary import option for transactions from CSV files (fallback when not using Google Sheets).
- **Merchant memory** — remembers which category you assigned to each merchant, so repeat merchants are auto-suggested.
- **Multi-session support** — save a partially-categorized batch and come back to it later.
- **Undo** — step back if you mis-categorize a transaction.
- **Skip** — skip transactions you want to deal with later.
- **Manage categories** — add and delete custom categories with emoji and grouping.
- **Clipboard export** — copies categorized transactions as tab-separated rows (Date, Payee, Category, Account, Amount) for pasting into Excel.
- **Dark theme** — designed for comfortable use on mobile.

---

## Tech Stack

- **Frontend:** Single-file HTML app using React 18 (via CDN, no build step). All components use `React.createElement` — no JSX.
- **Styling:** Inline styles with a consistent dark color palette (`#1B1F3B` background, `#6C63FF` accent).
- **Persistence:** `localStorage` for categories, merchant mappings, sessions, and the configured web app URL.
- **Backend:** Google Apps Script deployed as a web app. The Google Sheet serves as a persistent transaction log.
- **Hosting:** GitHub Pages (static site, zero cost).

---

## Project Structure

```
transaction-categorizer/
  index.html      # The entire app — HTML, CSS, and JS in one file
  README.md       # This file
```

The companion Apps Script lives in the Google Sheet and is not part of this repo. It provides these endpoints:

| Endpoint | Method | Description |
|----------|--------|-------------|
| `?action=parseAndFetch` | GET | Parses new emails from Gmail, then returns uncategorized transactions |
| `?action=peek` | GET | Returns uncategorized transactions (no email parsing) |
| `?action=all` | GET | Returns all transactions |
| `?action=uncategorized` | GET | Returns uncategorized transactions (default) |
| (body: `{ updates: [...] }`) | POST | Writes categories back to the Sheet |

---

## Setup

### Prerequisites

- A Gmail account with Scotiabank InfoAlert emails (direct or forwarded).
- A Google Sheet to serve as the transaction log.
- The Apps Script deployed as a **Web App** (Execute as: Me, Who has access: Anyone).

### Steps

1. **Create a Google Sheet** and copy the Sheet ID from the URL.
2. **Open Extensions > Apps Script**, paste the script below, and update `CONFIG.SHEET_ID`.
3. **Run `setupSheet()`** once to create headers and formatting.
4. **Deploy as web app** — Deploy > New Deployment > Web App > Execute as: Me, Who has access: Anyone. Copy the URL.
5. **Open the categorizer** — Go to [fahyad.github.io/transaction-categorizer](https://fahyad.github.io/transaction-categorizer/).
6. **Configure** — Tap the gear icon and paste your Apps Script web app URL.
7. **Fetch & categorize** — Tap "Fetch Transactions" to parse emails and pull uncategorized rows.
8. **Export** — When done, tap "Copy to Clipboard" and paste into your budget spreadsheet.

---

## Apps Script

The full Google Apps Script to paste into your Sheet's script editor. This handles email parsing, sheet storage, and the web app API.

### Configuration

Update `CONFIG.SHEET_ID` with your Google Sheet ID:

```javascript
const CONFIG = {
  SHEET_ID: 'YOUR_SHEET_ID_HERE',
  TRANSACTIONS_TAB: 'Transactions',
  LOG_TAB: 'Parse Log',
  GMAIL_QUERY: '(from:infoalerts@scotiabank.com OR subject:"on your credit account" OR subject:"on your debit account" OR subject:"on your chequing account" OR subject:"on your savings account") is:unread',
  PROCESSED_LABEL: 'BudgetBot/Processed',
  FAILED_LABEL: 'BudgetBot/ParseFailed',
  TIMEZONE: 'America/Edmonton',
};
```

### Key Functions

| Function | Purpose |
|----------|---------|
| `processNewAlerts()` | Searches Gmail for unread Scotiabank alerts, parses them, writes to Sheet |
| `parseScotiabankAlert(message)` | Extracts transaction data from email body using regex patterns |
| `doGet(e)` | Web app GET endpoint — parses emails (if `parseAndFetch`) then returns transactions |
| `doPost(e)` | Web app POST endpoint — writes categories back to the Sheet |
| `setupSheet()` | Run once to create headers and formatting |

### doGet Endpoint

The `doGet` function supports the `?action=parseAndFetch` parameter, which triggers `processNewAlerts()` before returning data. This is what the app uses so that a single "Fetch Transactions" tap parses new emails and returns results:

```javascript
function doGet(e) {
  const action = (e && e.parameter && e.parameter.action) || 'uncategorized';

  // When the app requests parseAndFetch, parse emails first
  if (action === 'parseAndFetch') {
    processNewAlerts();
  }

  const sheet = getOrCreateSheet(CONFIG.TRANSACTIONS_TAB);
  const lastRow = sheet.getLastRow();

  if (lastRow < 2) {
    return jsonResponse({ transactions: [], message: 'No transactions found' });
  }

  const data = sheet.getRange(2, 1, lastRow - 1, 11).getValues();

  const transactions = [];
  for (let i = 0; i < data.length; i++) {
    const row = data[i];
    const txn = {
      row: i + 2,
      date: row[0],
      timeET: row[1],
      amount: row[2],
      merchant: row[3],
      transactionType: row[4],
      accountType: row[5],
      account: row[6],
      category: row[7],
      status: row[8],
      subject: row[9],
      messageId: row[10]
    };

    if (action === 'all' || txn.status === 'uncategorized' || !txn.status) {
      transactions.push(txn);
    }
  }

  return jsonResponse({ transactions: transactions });
}
```

### Response Format

```json
{
  "transactions": [
    {
      "row": 2,
      "date": "2025-03-15",
      "timeET": "3:42 pm",
      "amount": 47.52,
      "merchant": "SAFEWAY #4821",
      "transactionType": "authorization",
      "accountType": "Credit",
      "account": "4537*****197****",
      "category": "",
      "status": "uncategorized",
      "subject": "Activity on your credit account",
      "messageId": "18e1a2b3c4d5e6f7"
    }
  ]
}
```

### POST Format

```json
{
  "updates": [
    { "row": 2, "category": "Groceries" },
    { "row": 3, "category": "Gas" }
  ]
}
```

> **Important:** After modifying the script, you must redeploy the web app (Deploy > Manage Deployments > Edit > New Version > Deploy) for changes to take effect.

---

## Default Categories

| Group | Categories |
|-------|-----------|
| Living | Groceries, Gas, Parking, House things |
| Nice Things | Date night, Saajidah, Fahyad |
| Fixed | Rent, Epcor, Phones, Subscriptions, Student loans |
| Travel | Big trips, Small trips |
| Money | Credit card, Line of credit, TFSA |
| Other | NDEB, Saajidah birthday, Income |

Categories are fully customizable from the Manage Categories screen within the app.
