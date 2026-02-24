# Linkdrop — Wallet Integration Overview

## What is Linkdrop?

Linkdrop enables sending crypto tokens via shareable URLs (claim links). A sender deposits tokens into an escrow smart contract and receives a claim link. The recipient opens the link and redeems tokens to their wallet — no need to know the recipient's address upfront.

**Supported tokens:** Native tokens (ETH, MATIC, AVAX, etc.), ERC20, ERC721, ERC1155
**Supported chains:** Base, Polygon, Optimism, Arbitrum, Avalanche (additional EVM chains can be added within days on request)

---

## What Linkdrop Provides

| Component | Description |
|-----------|-------------|
| **Backend infrastructure** | Fully hosted by Linkdrop — the wallet does not need to deploy or maintain any servers |
| **Escrow smart contracts** | Onchain contracts that hold tokens until claimed or refunded |
| **SDK** | Client-side library the wallet integrates to create and redeem claim links |
| **API key** | Provided by Linkdrop for API authentication |
| **Relayers** | Linkdrop sponsors all claim transactions — recipients don't pay gas and don't need to have any crypto to claim a link |
| **Dashboard (Web UI)** | Web interface at [dashboard.linkdrop.io](https://dashboard.linkdrop.io) for businesses to create claim links in bulk |

### SDK Availability

| Language | Status |
|----------|--------|
| TypeScript | Available — [linkdrop-sdk on npm](https://www.npmjs.com/package/linkdrop-sdk) (latest: 3.15.2-beta) |
| Go | Available |
| Swift (iOS) | Can be ported on request (~1 month) |
| Kotlin (Android) | Can be ported on request (~1 month) |

### Fees

- **Integration is free for wallets.** There are no costs to integrate or use the P2P send-via-link feature.
- Linkdrop charges fees only to Dashboard (B2B) users who create bulk claim links via the web UI.

---

## Link Creation Modes

There are two independent modes. Wallets can support both or just one.

### Mode 1: P2P Links (Send-via-Link)

A wallet user sends tokens to anyone by creating a shareable link. Each link has its own deposit transaction.

**Who creates links:** End users, directly in the wallet app.

**User flow — Sender:**

```
┌─────────────────────────────────────────────────────────────────┐
│ 1. User opens "Send via Link"                                   │
│ 2. Selects token, chain, and enters amount                      │
│ 3. Optionally adds a message (encrypted, max 140 chars)         │
│ 4. Confirms deposit (on-chain tx or gasless signature)          │
│ 5. Receives a claim link URL                                    │
│ 6. Shares the link (messaging apps, share sheet, copy/paste)    │
└─────────────────────────────────────────────────────────────────┘
```

*Screenshots from the Coinbase Wallet integration:*

| Select token & amount | Confirm deposit | Share link |
|:---:|:---:|:---:|
| <img src="images/sender-amount.png" width="200"> | <img src="images/sender-confirm.png" width="200"> | <img src="images/sender-share-link.png" width="200"> |

| OS share sheet | Link preview in Telegram |
|:---:|:---:|
| <img src="images/sender-share-sheet.png" width="200"> | <img src="images/sender-telegram-preview.png" width="200"> |

**User flow — Receiver:**

```
┌─────────────────────────────────────────────────────────────────┐
│ 1. Opens claim link (deep link / universal link into the app)   │
│ 2. Sees token, amount, chain, sender, and optional message      │
│ 3. Taps "Claim" → tokens are sent to their wallet address       │
└─────────────────────────────────────────────────────────────────┘
```

| Claim screen | Success |
|:---:|:---:|
| <img src="images/receiver-claim.png" width="200"> | <img src="images/receiver-success.png" width="200"> |

**Screens the wallet needs to build:**

| Screen | Description |
|--------|-------------|
| **Send-via-link** | Token/chain picker → amount input → optional message → deposit confirmation → share link |
| **Claim/redeem** | Display link details (token, amount, chain, sender, message) → claim button → success |
| **Transaction details** | Claim link status, deposit/redeem operations, timestamps, tx hashes |
| **Sender history** | List of all created claim links with current statuses |

| Transaction history | Transaction details |
|:---:|:---:|
| <img src="images/sender-tx-history.png" width="200"> | <img src="images/sender-tx-detail.png" width="200"> |

### Mode 2: Dashboard Links (Redeem Only)

Businesses and projects create claim links in bulk via the Linkdrop Dashboard web UI. The business approves the Linkdrop contract to transfer tokens on their behalf — tokens stay in the business's wallet until each link is individually claimed. Links are distributed to end users (e.g., via email campaigns, QR codes).

**Who creates links:** Businesses using the Linkdrop Dashboard. Not the wallet user.

**The wallet only needs to support the redeem side:**

**User flow — Receiver:**

```
┌─────────────────────────────────────────────────────────────────┐
│ 1. User receives a claim link (email, QR code, social media)    │
│ 2. Opens link → deep links into the wallet app                  │
│ 3. Sees token, amount, and chain                                │
│ 4. Taps "Claim" → tokens are sent to their wallet address       │
└─────────────────────────────────────────────────────────────────┘
```

**Screens the wallet needs to build:**

| Screen | Description |
|--------|-------------|
| **Claim/redeem** | Reuses the same redeem screen as P2P — no additional UI needed. The SDK detects the link type automatically. |

### Shared Redeem Flow

The claim/redeem screen is the same for both P2P and Dashboard links. The SDK detects the link type automatically and handles the differences internally. **You only need to build the redeem UI once** — it works for both modes out of the box.

### Comparing Modes

| | P2P Links | Dashboard Links |
|---|-----------|-----------------|
| **Effort** | 4 screens (send, claim, details, history) | 1 screen (claim) — reuses the same redeem screen as P2P |
| **Who creates links** | Wallet end users | Businesses via Dashboard web UI |
| **Value to users** | Send tokens to anyone without knowing their address | Receive tokens from campaigns, promotions, airdrops |
| **Depends on the other mode?** | No — fully independent | No — fully independent |

---

## How It Works (Simplified)

### P2P Flow

```
Sender (wallet user)            Linkdrop                         Receiver
  │                                │                                │
  │  1. Create claim link (SDK)    │                                │
  │ ─────────────────────────────> │                                │
  │  ← claim link object          │                                │
  │                                │                                │
  │  2. Deposit tokens to escrow   │                                │
  │ ─────────────────────────────> │                                │
  │  ← claim URL                  │                                │
  │                                │                                │
  │  3. Share URL  ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─>│
  │                                │                                │
  │                                │  4. Open URL, see link details │
  │                                │ <──────────────────────────────│
  │                                │  → token, amount, chain, etc.  │
  │                                │                                │
  │                                │  5. Claim to wallet address    │
  │                                │ <──────────────────────────────│
  │                                │  → tokens sent to receiver     │
```

**Lifecycle:** created → depositing → deposited → redeemed / refunded / cancelled

- **deposited** — tokens locked in escrow, link is claimable
- **redeemed** — receiver claimed the tokens
- **refunded** — sender reclaimed after expiration (default: 15 days)
- **cancelled** — sender cancelled before claim

### Dashboard Flow

```
Business (Dashboard UI)         Linkdrop                         Receiver
  │                                │                                │
  │  1. Create links in bulk       │                                │
  │ ─────────────────────────────> │                                │
  │                                │                                │
  │  2. Approve tx (tokens stay    │
  │     in business's wallet)      │                                │
  │ ─────────────────────────────> │                                │
  │  ← claim URLs (CSV download)  │                                │
  │                                │                                │
  │  3. Distribute URLs ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─>│
  │     (email, QR, social, etc.)  │                                │
  │                                │                                │
  │                                │  4. Open URL, see link details │
  │                                │ <──────────────────────────────│
  │                                │  → token, amount, chain, etc.  │
  │                                │                                │
  │                                │  5. Claim to wallet address    │
  │                                │ <──────────────────────────────│
  │                                │  → tokens sent to receiver     │
```

Dashboard links arrive already claimable — there is no deposit step to wait for. The SDK returns the same normalized statuses for both link types (`deposited`, `redeemed`, `cancelled`, etc.), so the wallet displays them the same way.

Note: Steps 4–5 (the redeem flow) are identical for both P2P and Dashboard links. The SDK detects the link type automatically — the wallet only needs to build the redeem UI once.

---

## Demos & Screenshots

Screenshots from the Coinbase Wallet integration are embedded in the flow sections above.

**Screen recordings (Coinbase Wallet):**
- [Sender flow](https://www.youtube.com/watch?v=YaVMo_PiQwY) — creating and sharing a claim link
- [Recipient flow](https://www.youtube.com/watch?v=MEUcuJQGLbg) — opening a link and claiming tokens

---

## FAQ

**Do we need to deploy or host anything?**
No. Linkdrop hosts all backend infrastructure. The wallet only integrates the SDK client-side.

**Can we support additional EVM chains?**
Yes. Linkdrop can add support for any EVM chain within a couple of days on request.

**What if the sender loses the claim URL?**
The sender can regenerate it from the wallet (requires a signature). Claim URLs are never stored server-side.

**What happens if a link expires unclaimed?**
Tokens are automatically refundable to the sender after expiration.

---

## Resources

- **Technical Reference:** [wallet-integration-technical.md](wallet-integration-technical.md) — architecture, security model, SDK code examples, API details
- **SDK repository:** [github.com/LinkdropHQ/linkdrop-sdk](https://github.com/LinkdropHQ/linkdrop-sdk)
- **Demo recordings:** [Sender flow](https://www.youtube.com/watch?v=YaVMo_PiQwY) · [Recipient flow](https://www.youtube.com/watch?v=MEUcuJQGLbg)

---

**Contact:** hi@linkdrop.io
