# Transaction Categorizer

A mobile-friendly web app for categorizing Scotiabank transactions and exporting them to a budget spreadsheet. Hosted on GitHub Pages, it connects to a Google Sheet via a Google Apps Script web app to fetch uncategorized transactions, let you assign categories, and write the results back.

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
  Google Apps Script (ScotiabankParser.gs)
    - Parses email body with regex
    - Writes transactions to Google Sheet
    - Serves uncategorized rows via web app API
        |
        v
  Transaction Categorizer (this app)
    - Fetches uncategorized transactions from the Sheet
    - Presents a swipe-through UI for assigning categories
    - POSTs category updates back to the Sheet
    - Copies formatted output to clipboard for pasting into Excel
```

### Data Flow

1. **Scotiabank** sends transaction alert emails (InfoAlerts) to your email.
2. Emails are forwarded to **Gmail**, where a bound **Google Apps Script** parses them and logs each transaction as a row in a Google Sheet.
3. This **Transaction Categorizer** app calls the Apps Script `doGet` endpoint to fetch all uncategorized rows.
4. You tap through each transaction and assign a category (Groceries, Gas, Rent, etc.).
5. The app calls the Apps Script `doPost` endpoint to write the chosen category back to the Sheet.
6. A **Copy to Clipboard** button formats the categorized transactions as tab-separated text, ready to paste into your Excel budget workbook.

---

## Features

- **Google Sheets integration** — reads and writes directly via Apps Script web app (no API keys or OAuth flow needed).
- **One-tap categorization** — transactions appear one at a time with a grid of category buttons.
- **Merchant memory** — remembers which category you assigned to each merchant, so repeat merchants are auto-suggested.
- **Multi-session support** — save a partially-categorized batch and come back to it later.
- **Undo** — step back if you mis-categorize a transaction.
- **Skip** — skip transactions you want to deal with later.
- **Manage categories** — add, edit, reorder, and delete categories with custom emoji and grouping.
- **Clipboard export** — copies categorized transactions as tab-separated rows (Date, Amount, Merchant, Category) for pasting into Excel.
- **Dark theme** — designed for comfortable use on mobile.

---

## Tech Stack

- **Frontend:** Single-file HTML app using React 18 (via CDN, no build step). All components use `React.createElement` — no JSX.
- **Styling:** Inline styles with a consistent dark color palette (`#1B1F3B` background, `#6C63FF` accent).
- **Persistence:** `localStorage` for categories, merchant mappings, sessions, and the configured web app URL.
- **Backend:** Google Apps Script deployed as a web app. No server, no database — the Google Sheet is the single source of truth.
- **Hosting:** GitHub Pages (static site, zero cost).

---

## Project Structure

```
transaction-categorizer/
  index.html      # The entire app — HTML, CSS, and JS in one file
  README.md       # This file
```

The companion Apps Script (`ScotiabankParser.gs`) lives in the Google Sheet and is not part of this repo. It provides two endpoints:

| Endpoint | Method | Description |
|----------|--------|-------------|
| `?action=uncategorized` | GET | Returns uncategorized transactions as JSON |
| `?action=all` | GET | Returns all transactions as JSON |
| (body: `{ updates: [...] }`) | POST | Writes categories back to the Sheet |

---

## Setup

### Prerequisites

- A Gmail account with Scotiabank InfoAlert emails (direct or forwarded).
- A Google Sheet with the `ScotiabankParser.gs` script installed and the `setupSheet()` function run once.
- The Apps Script deployed as a **Web App** (Execute as: Me, Who has access: Anyone).

### Steps

1. **Deploy the parser** — In your Google Sheet, open Extensions > Apps Script, paste `ScotiabankParser.gs`, and deploy as a web app. Copy the web app URL.
2. **Open the categorizer** — Go to [fahyad.github.io/transaction-categorizer](https://fahyad.github.io/transaction-categorizer/).
3. **Configure** — Tap the gear icon (Settings) and paste your Apps Script web app URL.
4. **Fetch & categorize** — Tap "Fetch Transactions" to pull uncategorized rows, then tap categories to assign them.
5. **Export** — When done, tap "Copy to Clipboard" and paste into your budget spreadsheet.

---

## Apps Script Endpoints

The `doGet` response returns:

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

The `doPost` request expects:

```json
{
  "updates": [
    { "row": 2, "category": "Groceries" },
    { "row": 3, "category": "Gas" }
  ]
}
```

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
