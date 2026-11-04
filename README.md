# Drip

Subscription tracker built for Canadians. Find what you're paying for via Gmail scan, CSV upload, or live bank link.

> **Public showcase repo** — architecture and design only. Source lives in a private repo.

## Stack

| Layer | Tech |
|-------|------|
| **drip-web** | Next.js 16 · React 19 · Tailwind CSS 4 · PWA (installable on iOS/Android) |
| **drip-api** | Bun · Hono · Drizzle ORM · SQLite |
| **Auth** | Google OAuth (Gmail), Plaid Link (banks) |
| **AI** | Claude Haiku 4.5 for unknown-service enrichment |

## Architecture

```mermaid
flowchart LR
    User([User / Browser])

    subgraph Web["drip-web :3002"]
        UI[Next.js Dashboard]
        Modals["📧 Gmail · 💳 CSV · 🏦 Bank modals"]
    end

    subgraph API["drip-api :3001 (Bun + Hono)"]
        Routes["/api/* routes"]
        Detect["Recurrence detector<br/>(shared by CSV + Plaid)"]
        Catalog["Service catalog<br/>+ Claude enrichment"]
    end

    DB[("SQLite<br/>drip.db")]
    Gmail[Google OAuth<br/>+ Gmail API]
    Claude[Anthropic API<br/>Haiku 4.5]
    Plaid[Plaid API<br/>Sandbox/Dev]

    User --> UI
    UI --> Modals
    Modals -->|fetch| Routes
    Routes --> DB
    Routes --> Detect
    Routes --> Catalog
    Catalog --> Claude
    Routes -.OAuth + scan.-> Gmail
    Routes -.link + transactions.-> Plaid
```

## Discovery flows

Three import paths, all funnel into the same review-and-import UI:

| Source | How it works | Confidence signal |
|--------|--------------|-------------------|
| **📧 Gmail** | OAuth read-only; scans last 12 months of billing emails; matches sender domain against a curated service catalog (Netflix, Spotify, Apple, etc.). Unknown senders go to Claude Haiku for one-shot identification. | `high` if catalog hit · `medium`/`low` if AI-enriched |
| **💳 CSV upload** | Drop a bank/credit-card statement (RBC, TD, Amex, Capital One, Scotia). Auto-detects column headers, normalizes merchant strings, groups by merchant, detects monthly/weekly/yearly cadence (±15% amount tolerance, ≥2 charges). | `high` if catalog hit · `medium` if 3+ occurrences |
| **🏦 Plaid** | Plaid Link → access token → 12mo transaction pull → same recurrence detector as CSV. Sandbox tier is free; major CA banks supported. | Same as CSV |

### 📧 Gmail

```mermaid
sequenceDiagram
    actor U as User
    participant W as drip-web
    participant A as drip-api
    participant G as Google
    participant C as Claude

    U->>W: Click "Connect Gmail"
    W->>A: GET /api/auth/google
    A-->>U: 302 → Google consent
    U->>G: Approve scopes
    G-->>A: GET /callback?code=...
    A->>G: Exchange code → tokens
    A->>A: Store in oauth_tokens
    A-->>W: 302 → ?gmail_connected=true
    W->>A: POST /api/email/scan
    A->>G: gmail.messages.list (12mo billing query)
    loop each message
        A->>A: matchService(domain, subject)
        opt unknown sender
            A->>C: enrichUnknownService()
        end
    end
    A-->>W: detected subscriptions
    U->>W: Select + Import
    W->>A: POST /api/subscriptions (× N)
```

### 💳 CSV upload

```mermaid
sequenceDiagram
    actor U as User
    participant W as drip-web
    participant A as drip-api

    U->>W: Pick CSV file
    W->>W: file.text()
    W->>A: POST /api/cards/scan (raw CSV)
    A->>A: parseCsv → normalize merchants
    A->>A: group + findRecurring (±15%, cadence)
    A->>A: matchService per group
    A-->>W: detected subscriptions
    U->>W: Select + Import
    W->>A: POST /api/subscriptions (× N)
```

### 🏦 Bank link (Plaid)

```mermaid
sequenceDiagram
    actor U as User
    participant W as drip-web
    participant A as drip-api
    participant P as Plaid

    W->>A: POST /api/plaid/link-token
    A->>P: linkTokenCreate
    P-->>A: link_token
    A-->>W: link_token
    U->>W: Click "Connect Bank"
    W->>P: Plaid Link UI (bank login)
    P-->>W: public_token (on success)
    W->>A: POST /api/plaid/exchange
    A->>P: itemPublicTokenExchange
    P-->>A: access_token
    A->>A: Store in oauth_tokens
    W->>A: POST /api/plaid/scan
    A->>P: transactionsGet (12mo, paginated)
    A->>A: findRecurring (same detector as CSV)
    A->>A: matchService per group
    A-->>W: detected subscriptions
    U->>W: Select + Import
    W->>A: POST /api/subscriptions (× N)
```

## Why I built this

Personal subscription sprawl is real. Existing trackers either want bank credentials I'd rather not hand over, or are manual-entry only. Drip lets you pick the comfort level:

- **Most private**: manual entry or CSV upload (no third party touches your data)
- **Most convenient**: Plaid bank link (read-only, no money movement)
- **In-between**: Gmail scan (read-only inbox access, billing keywords only)

## Author

[Nick Chung](https://github.com/NickChunglolz)
