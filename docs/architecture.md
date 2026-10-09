# Architecture — Stellar MicroPay

## Overview

Stellar MicroPay is a three-tier Web3 application:

```
┌─────────────────────────────────────────────────────────────────┐
│                         User's Browser                          │
│                                                                 │
│   ┌──────────────────────┐    ┌────────────────────────────┐   │
│   │   Next.js Frontend   │    │    Freighter Extension     │   │
│   │  (React + Tailwind)  │◄──►│   (Stellar Wallet)         │   │
│   └──────────┬───────────┘    └────────────────────────────┘   │
└──────────────┼──────────────────────────────────────────────────┘
               │ HTTP (REST)
               ▼
┌──────────────────────────┐
│   Node.js Backend API    │
│   (Express)              │
│                          │
│  • Account lookups       │
│  • Payment history       │
│  • Username resolution   │
└──────────────┬───────────┘
               │ Horizon REST API
               ▼
┌──────────────────────────┐       ┌──────────────────────────┐
│   Stellar Horizon API    │◄─────►│   Stellar Network        │
│   (horizon-testnet       │       │   (Validators)           │
│    .stellar.org)         │       │                          │
└──────────────────────────┘       └──────────────────────────┘
                                              ▲
                                              │ Soroban
┌──────────────────────────┐       ┌──────────────────────────┐
│   Soroban Smart Contract │       │   Turrets Sidecar        │
│   (Rust/WASM)            │       │   (Port 4100)            │
│                          │       │                          │
│  • Streaming payments    │◄─────►│  Off-chain Turrets       │
│    (open/claim/top-up/   │       │  compute & signing       │
│     close + pause/resume)│       └──────────────────────────┘
│  • Escrow payments (v2.1)│
│  • Milestone escrow with │       ┌──────────────────────────┐
│    dispute timeout (v2.1)│       │   Redis Cache            │
│  • Creator tipping       │       │   (Port 6379)            │
│    (v1.4)                │       │                          │
│  • Micro-transaction     │       │  Hot-path caching        │
│    batching (v2.0)       │       └──────────────────────────┘
│  • NFT payment receipts  │
│    (v1.5)                │
└──────────────────────────┘

## Stream Lifecycle

The streaming payment contract exposes a full lifecycle state machine
(`contracts/stellar-micropay-contract/src/lib.rs`):

1. **open** — sender initializes a stream to a recipient with a total
   amount and duration; funds are locked in the contract escrow.
2. **claim** — the recipient (or anyone on their behalf) withdraws the
   vested amount accrued since the last claim, as often as desired.
3. **top-up** — the sender adds funds to an existing stream, extending
   either the duration or the per-second rate.
4. **pause** — the sender temporarily halts vesting (dispute window).
5. **resume** — vesting continues after a pause.
6. **close** — either party ends the stream; unvested funds return to
   the sender, vested funds remain claimable by the recipient.

Escrow payments (ROADMAP v2.1) add milestone-based release with a
dispute timeout on top of this lifecycle; creator tipping (v1.4),
micro-transaction batching (v2.0) and NFT payment receipts (v1.5) are
adjacent capabilities recorded in the same ledger.
```

## Payment Flow

```
User fills form ──► Build TX (Stellar SDK) ──► Sign (Freighter)
                                                      │
                                                      ▼
View on Explorer ◄── Success ◄── Submit to Horizon Network
```

### Step-by-step

1. **User inputs** destination address, amount, optional memo
2. **Frontend** builds an unsigned Stellar transaction using `stellar-sdk`
3. **Freighter** prompts the user to review and sign the transaction
4. **Frontend** submits the signed XDR to Stellar's Horizon API
5. **Horizon** broadcasts the transaction to the Stellar validator network
6. **Network** confirms the transaction in 3–5 seconds
7. **Frontend** polls or receives the confirmed transaction hash

## Key Design Decisions

### Non-custodial
Private keys never leave the user's device. Freighter handles all signing locally in the browser extension.

### Client-side transactions
Transaction building and submission happen directly in the browser via the Stellar SDK. The backend is not required for core payment functionality — this reduces attack surface.

### Backend as optional enhancement
The Node.js backend provides:
- Username-to-address resolution (future)
- Cached payment history for performance
- Analytics and aggregation

If the backend is down, users can still send payments via the frontend.

### Testnet first
The default configuration targets Stellar Testnet. Switching to Mainnet requires only an environment variable change.

## Security Considerations

| Concern | Mitigation |
|---------|-----------|
| Private key exposure | Freighter handles signing — keys never touch the app |
| Transaction replay | Stellar sequence numbers prevent replays |
| Man-in-the-middle | HTTPS enforced; Horizon API uses TLS |
| Malicious destination | User confirms destination in Freighter before signing |
| Rate abuse | Express rate limiter on backend API |
| XSS | React's default escaping; no `dangerouslySetInnerHTML` |

## File Dependency Map

```
pages/
  _app.tsx          ← Global wallet state (publicKey)
  index.tsx         ← Landing page + WalletConnect
  dashboard.tsx     ← Balance + SendPaymentForm + TransactionList
  transactions.tsx  ← Full TransactionList

components/
  WalletConnect.tsx ← Uses lib/wallet.ts
  SendPaymentForm.tsx ← Uses lib/stellar.ts + lib/wallet.ts
  TransactionList.tsx ← Uses lib/stellar.ts
  Navbar.tsx         ← Uses lib/stellar.ts (shortenAddress)

lib/
  stellar.ts  ← Horizon API calls, TX building, TX submission
  wallet.ts   ← Freighter integration (connect, sign)

utils/
  format.ts   ← XLM formatting, date formatting, clipboard
```
