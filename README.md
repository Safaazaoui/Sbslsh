# 🍊 Tangerine Subscription Detector Agent

> An AI-powered agent built into the Tangerine mobile app that detects recurring charges, scores them by usage, and cancels unused subscriptions in one tap.

Built at **Hackathon 2026** 

---

## The Problem

Tangerine has 2M+ clients with no way to detect unused subscriptions or help them cancel. The average Canadian wastes **$97/month** — **$1,164/year** — on forgotten charges. No Canadian bank currently offers a native solution.

## The Solution

The Subscription Detector Agent runs in the background against a user's Tangerine transaction history. It identifies recurring charges, evaluates whether each subscription is being actively used, and enables one-tap cancellation — without the user ever leaving the app.

| Metric | Value |
|---|---|
| Avg. monthly waste detected | $97 |
| Avg. annual savings potential | $1,164 |
| Tangerine clients | 2M+ |
| Avg. subscriptions per user | 11 |

---

## How It Works

### 1. Subscription Detection
Scans transaction history for charges repeating on a 30-day or 365-day cadence. Groups by normalized merchant name and amount. Flags any pattern appearing 2+ times as a subscription.

### 2. Last-Activity Detection
Connects to Gmail (read-only OAuth) to find the most recent activity email from each subscription service. Last email date = proxy for last use. No passwords, no scraping.

### 3. AI-Powered Verdict
Claude AI receives the usage data and generates a plain-language recommendation for each subscription: **CANCEL**, **REVIEW**, or **KEEP** — with specific reasoning using the user's real numbers.

### 4. One-Tap Cancellation
The user reviews the AI recommendation and taps Approve. The agent sends a cancellation email via Gmail API or navigates a cancellation flow automatically. No AI is involved in the actual cancellation step.

### 5. Post-Cancel Monitoring
After cancellation, the agent watches for future charges from the same merchant and alerts the user immediately if one appears — protecting against accidental re-billing.

---

## Sample Output

| Service | Monthly Cost | Last Active | Verdict |
|---|---|---|---|
| Netflix | $20.99/mo | 3 days ago | ✅ KEEP |
| Spotify | $11.99/mo | Today | ✅ KEEP |
| LinkedIn Premium | $49.99/mo | 45 days ago | ❌ CANCEL |
| Duolingo Plus | $9.99/mo | 73 days ago | ❌ CANCEL |
| Canva Pro | $16.99/mo | 23 days ago | ⚠️ REVIEW |

**Total potential savings: $76.97/month · $923.64/year**

---

## AI vs. Hardcoded Logic

A deliberate engineering choice: use rules wherever possible, AI only where rules genuinely can't work.

**Hardcoded (rules + math) — zero API cost, instant, fully auditable:**
- Detecting recurring date patterns
- Grouping merchants by name + amount
- Scoring usage by days since last active
- CANCEL / REVIEW / KEEP verdict logic
- Sending cancellation emails via Gmail API
- Plan tier comparison lookups

**AI only — fires only on user interaction (~$0.01/session):**
- Labelling unknown merchant names (e.g. `FGLD*SRVCS 8882341122`)
- Writing plain-language explanations on the detail screen
- Answering conversational follow-up questions ("what do I lose if I cancel X?")

> **The one-liner:** Rules find the problem. AI explains it and talks to you about it. The code cancels it.

---

## Technical Architecture

### Data Sources

| Source | Details |
|---|---|
| Tangerine Transaction Data | Chequing + credit card history — available natively. Fields: date, merchant name, amount, account type. |
| Gmail OAuth (read-only) | Finds the most recent activity email per subscription. Scope: `gmail.readonly` only. Never sends without explicit user approval. |
| Subscription Plan Database | A maintained JSON file mapping merchant names to plan tiers, pricing, and free alternatives. |

### Detection Algorithm

1. **Normalize merchant names** — strip payment processor prefixes (`SQ*`, `TST*`, `SP*`), location suffixes, and store numbers.
2. **Group by merchant + amount** — group transactions by normalized name and amount (±$0.50 tolerance for tax variations).
3. **Check recurrence interval** — calculate gaps between consecutive charges. Gaps clustering around 30 days (±5) or 365 days (±10) = subscription confirmed.
4. **Score and verdict** — apply rules: CANCEL if last active >45 days; CANCEL if annual renewal <14 days AND unused >30 days; REVIEW if 20–45 days; KEEP if <20 days.

### Cancellation Methods

**Method 1 — Email cancellation**
Agent composes a formal cancellation email and sends it via Gmail API. User reviews and approves before anything is sent. Works for: LinkedIn, Adobe, Dropbox, most SaaS services.

**Method 2 — Browser automation**
A Playwright/Puppeteer script navigates to the service's cancellation page and clicks through the flow automatically. User taps Approve once. Works for: Netflix, Spotify, Duolingo, Canva.

---

## Setup (Backboard.io)

Build in this exact order:

### Step 1 — API Keys
In the Backboard sidebar, go to **API Keys → Add Key** and paste your Anthropic API key from [console.anthropic.com](https://console.anthropic.com). Use model `claude-haiku-4-5` for lowest cost.

### Step 2 — Documents
Upload two files to your assistant:
- `transactions.csv` — your Tangerine transaction export (last 6 months)
- `subscriptions.txt` — the plan knowledge base (see Appendix below)

### Step 3 — OAuth Apps (Gmail)
In **OAuth Apps → New App**, select Google, set scope to `gmail.readonly`, and follow the Google Cloud Console setup to get your Client ID and Secret.

> **Hackathon shortcut:** Skip Gmail OAuth and add last-activity dates manually in Memories instead.

### Step 4 — Memories
Add two memories to give the agent personalized context:

**Memory 1 — Last activity dates:**
```
User subscription last activity:
- Netflix: used 3 days ago
- Spotify: used today
- LinkedIn: last activity 51 days ago — no logins detected
- Duolingo: last activity 73 days ago — no lessons completed
- Canva: last activity 23 days ago — 2 designs exported
```

**Memory 2 — User profile:**
```
User profile:
- Bank: Tangerine
- Monthly subscription spend: $284
- Total subscriptions detected: 11
- Card on file: ends 4821
```

### Step 5 — Chat (the agent)
Create a new assistant named **Tangerine Subscription Agent** with this system prompt:

```
You are a subscription detection agent for Tangerine Bank.
You have access to the user's bank transactions (uploaded CSV document) and their subscription last-activity data (in Memories).

When the user says "analyze my subscriptions":
1. Read the uploaded CSV document
2. Find all charges that repeat monthly or annually
3. Cross-reference last activity from Memories
4. Output a table: Service | Monthly Cost | Last Active | Verdict | Reason
5. End with: Total you could save: $X/month

Verdict rules:
- CANCEL if last active > 45 days ago
- CANCEL if annual renewal within 14 days AND last active > 30 days
- REVIEW if last active 20-45 days ago
- KEEP if last active < 20 days

For CANCEL always state: the free alternative and exact annual savings.
For follow-up questions answer using the user's real numbers only. Never give generic advice.
```

### Test Messages (run in order)
1. `analyze my subscriptions`
2. `what do I lose if I cancel LinkedIn?`
3. `cancel LinkedIn for me`
4. `how much would I save cancelling everything flagged?`

---

## Verdict Rules Reference

| Verdict | Condition | Action |
|---|---|---|
| ❌ CANCEL | Last active > 45 days | Show free alternative + annual savings |
| ❌ CANCEL (urgent) | Annual renewal < 14 days AND last active > 30 days | Show renewal date prominently |
| ⚠️ REVIEW | Last active 20–45 days | Prompt user to decide |
| ✅ KEEP | Last active < 20 days | No action needed |

---

## License

Built for Hackathon 2026. Internal Tangerine prototype.
