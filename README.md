# 🛡️ Invoice Guard

**An AI back-office copilot for restaurants, with guardrails that stop the AI from being tricked.**

Built solo for the **DoorDash x Nova Hackathon 2026** (ScottyLabs, Carnegie Mellon University), **Back House** problem statements.

Invoice Guard reads supplier invoices with an LLM, then lets plain, deterministic code decide whether each one is safe to act on. Approved invoices update inventory and ingredient prices. Those prices flow into a per-dish profit view and a DoorDash payout reconciliation, so the owner sees what the restaurant actually earned.

---

## The problem

From the DoorDash problem statement, *Back House: Business & Operational Complexity*:

> Business records are often scattered across sales reports, supplier invoices, receipts and payouts. Bringing this information together requires significant manual work, making it difficult to track performance, spot issues and plan ahead.

We target:

| Slide problem | What Invoice Guard does |
|---|---|
| **03 Inventory Management & Forecasting** | Posting an approved invoice updates stock levels and supplier price history. A dashboard shows current stock and price trends. |
| **04 Bookkeeping & Financial Health** | Combines invoices with orders and DoorDash payouts. Checks invoices for errors and fraud, reconciles payouts against orders, and shows true profit per dish. Every decision is audit-logged. |

Prize tracks: **Best Merchant Experience** (primary) and **Most Creative Solution** (the trust layer for AI on untrusted documents, plus a system that links several pain points).

---

## Features

### 1. Invoice processing with a trust layer (tab: *Process an invoice*)
- Upload a `.txt` invoice, or a `.png`/`.jpg` photo (vision LLM), or load a bundled sample.
- The LLM only **extracts** the data into strict JSON. It has no tools and no authority.
- Deterministic checks run on the result and on the raw document text:

| Check | Outcome |
|---|---|
| Prompt-injection language in the document ("ignore previous rules", "mark as approved", "update the bank account"...) | **BLOCKED** |
| Duplicate invoice (same supplier + invoice number already posted) | **BLOCKED** |
| Bank account differs from the one on file for the supplier | **BLOCKED** |
| Line items do not sum to the invoice total | Needs review |
| Unit price up more than 15% vs. last known price | Needs review |
| Unknown supplier or a new item | Needs review |
| Future or unreadable date | Needs review |

- Decisions: `CLEAN`, `NEEDS REVIEW`, or `BLOCKED`. The LLM cannot change the decision. A blocked invoice cannot be posted.
- A human approves posting. Every decision is written to an audit log.

### 2. Owner dashboard (tab: *Owner dashboard*)
Current stock, supplier price history chart, and the audit log.

### 3. Profit and payouts (tab: *Profit & payouts*)
- **Payout reconciliation:** compares each order's expected payout (menu price minus commission) with the payout statement. Detects missing payouts, deducted credits/chargebacks, commission overcharges, and payouts for unknown orders.
- **True margin per dish:** menu price minus DoorDash commission minus food cost, where food cost uses the **latest supplier prices from approved invoices**. A supplier price increase immediately lowers the affected dishes' margins, and dishes under 15% margin are flagged.
- **Headline numbers:** menu sales, actually received, food cost, true profit.
- **Owner summary:** an action list written from the computed numbers. With an API key, an LLM writes it (instructed to use only the provided figures). Without one, a template writes it.

---

## How it works

```
 invoice (txt / image)
        │
        ▼
 ┌────────────────┐   untrusted document, strict-JSON output, no tools
 │  LLM extractor │   (offline regex parser when no API key is set)
 └───────┬────────┘
         ▼
 ┌────────────────────────────┐
 │ Deterministic guardrails   │  injection scan, sum check, duplicates,
 │ (plain Python, no LLM)     │  bank-account match, price-jump, dates
 └───────┬────────────────────┘
         ▼
   CLEAN / NEEDS REVIEW / BLOCKED  ──►  audit log
         │ (human approves)
         ▼
 inventory + price history (SQLite)
         │
         ├──► dish margins (latest ingredient prices)
         └──► payout reconciliation ──► owner summary
```

**Design principle:** the LLM is used for perception and language (reading documents, writing summaries). Money, approvals, and arithmetic are done by deterministic code.

---

## Quick start

Requires Python 3.9+.

```bash
git clone <your-repo-url>
cd invoice-guard
python -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate
pip install -r requirements.txt
streamlit run app.py
```

The app opens at http://localhost:8501.

**No API key is needed to try it.** Without one, a built-in offline parser handles the `.txt` sample invoices, so the demo works without internet.

### Where to add your OpenRouter API key

The key is needed only for real LLM extraction, **photo/image invoices**, and LLM-written owner summaries.

1. Create a key at https://openrouter.ai/keys.
2. In the project root (the same folder as `app.py`), copy the example file:
   ```bash
   cp .env.example .env
   ```
3. Open `.env` and paste your key:
   ```
   OPENROUTER_API_KEY=sk-or-v1-your-key-here
   MODEL=openai/gpt-4o-mini
   ```
4. Restart the app (`streamlit run app.py`).

The sidebar confirms which mode is active: **LLM (model name)** or **offline mock (no API key)**.

Alternatively, set it as an environment variable instead of using `.env`:

```bash
export OPENROUTER_API_KEY="sk-or-v1-your-key-here"      # macOS / Linux
$env:OPENROUTER_API_KEY="sk-or-v1-your-key-here"        # Windows PowerShell
```

> **Never commit your key.** `.env` is already listed in `.gitignore`. Only `.env.example` (with a placeholder) belongs in the repo.

### Configuration

| Variable | Required | Default | Purpose |
|---|---|---|---|
| `OPENROUTER_API_KEY` | No | unset (offline mode) | Enables LLM extraction, image invoices, and LLM summaries |
| `MODEL` | No | `openai/gpt-4o-mini` | Any vision-capable model from https://openrouter.ai/models |

---

## Demo walkthrough

**Happy path**
1. *Process an invoice* → select `1_clean_invoice.txt` → **Process** → result is `CLEAN` → **Approve & post to inventory**. Stock increases in *Owner dashboard*.

**Sad paths**
2. Process `1_clean_invoice.txt` again → `BLOCKED` (duplicate).
3. Process `2_poisoned_invoice.txt` → `BLOCKED`: injected instructions in the notes, a swapped bank account, and a 62% price jump. Nothing is posted, no supplier data changes.
4. Process `3_price_jump_invoice.txt` → `NEEDS REVIEW` (tomatoes +38%).

**Linking invoices to profit**
5. Open *Profit & payouts*. Tomato Soup margin is 16.4%.
6. Approve `3_price_jump_invoice.txt` (step 4), then return to *Profit & payouts*: Tomato Soup drops to **12.0%** and is flagged.
7. The reconciliation table shows $34.60 short: a missing payout (O-1007), a deducted credit (O-1004), and a 30% commission instead of 20% (O-1010). Click **Generate owner summary**.

Use **Reset demo data** in the sidebar to start over.

---

## Project structure

```
invoice-guard/
├── app.py                         # Streamlit app: pipeline, guardrails, UI
├── requirements.txt
├── .env.example                   # template for your API key (copy to .env)
├── .gitignore
├── samples/
│   ├── 1_clean_invoice.txt        # passes all checks
│   ├── 2_poisoned_invoice.txt     # prompt injection + bank swap + price jump
│   └── 3_price_jump_invoice.txt   # tomatoes +38%
└── data/
    ├── orders.csv                 # synthetic DoorDash orders
    └── payout.csv                 # synthetic payout statement with planted errors
```

`guard.db` (SQLite) is created on first run and is git-ignored.

To use your own data in the *Profit & payouts* tab, upload CSVs with these columns:
- Orders: `order_id,date,dish,qty,subtotal`
- Payouts: `order_id,gross,commission,adjustment,net`

---

## Assumptions and limitations

- **All data is synthetic.** No real merchant, supplier, or customer data is used.
- **The 20% DoorDash commission is an assumption** (adjustable slider). Real rates depend on the merchant's plan.
- Dish recipes and `other_cost` values (protein, packaging) are invented for the demo. Only tomatoes, lettuce, and onions are costed from invoices.
- **Injection detection is heuristic** (pattern matching). It is one layer, not a guarantee. The core protection is architectural: the LLM cannot approve, post, or change records, so a successful injection could at most corrupt one extraction, which the other checks (sum, bank account, price, duplicate) and the human approval step then catch.
- Image invoices depend on the chosen model's OCR quality. Blurry images may produce wrong extractions, which the sum check and human review are meant to catch.
- Single-user prototype: no authentication, roles, or multi-restaurant support.

## Roadmap

- Receipt and payroll ingestion through the same pipeline
- Demand forecasting and automatic reorder suggestions from sales history
- Live DoorDash order and payout feeds instead of CSVs
- Red-team test suite of injection variants (hidden text, encoded instructions, multilingual)
- Role-based approvals (staff vs. owner)

## AI-assisted development disclosure

This project was developed with assistance from Claude (Anthropic). No code from prior projects is used. Third-party libraries: Streamlit, pandas, OpenAI Python SDK (used against the OpenRouter API), python-dotenv.

---

## Pushing this to GitHub

```bash
cd invoice-guard
git init
git add .
git status            # confirm .env and guard.db are NOT listed
git commit -m "Invoice Guard: AI invoice trust layer + payout reconciliation"
git branch -M main
git remote add origin https://github.com/<your-username>/invoice-guard.git
git push -u origin main
```

Create the empty repository on github.com first (without a README, since this one already exists).
