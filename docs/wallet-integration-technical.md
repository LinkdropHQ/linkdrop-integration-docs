# Linkdrop — Technical Reference

For a high-level overview of flows, screens, and integration steps, see [wallet-integration-overview.md](wallet-integration-overview.md).

---

## How Claim Links Work Under the Hood

### P2P Links

**Create & Deposit:**

1. SDK generates an secp256k1 keypair (standard Ethereum keypair). The private key becomes the claim code — the secret embedded in the claim URL that authorizes redemption.
2. SDK calls the Linkdrop API to register the transfer and receive fee information.
3. Sender deposits tokens to the escrow smart contract (on-chain transaction or gasless EIP-3009 signature for supported stablecoins).
4. SDK encodes the claim code, sender signature, transfer ID, chain ID, and version into a URL — the claim link.
5. The claim URL is returned to the sender to share. It is never stored server-side.

**Redeem:**

1. SDK parses the claim URL to extract the claim code and transfer metadata.
2. SDK calls the P2P API to fetch link details (token, amount, chain, status, etc.).
3. SDK generates a receiver signature from the claim code, proving the receiver possesses the link secret.
4. SDK calls the API `/redeem` endpoint with the receiver's wallet address and signature.
5. Linkdrop's relayer submits the on-chain redeem transaction. The receiver pays no gas.

**Lifecycle:**

```
 ┌──────────┐     ┌────────────┐     ┌───────────┐     ┌──────────┐
 │  created  │────>│ depositing │────>│ deposited │────>│ redeemed │
 └──────────┘     └────────────┘     └───────────┘     └──────────┘
                                           │
                                           ├───────────>┌──────────┐
                                           │            │ refunded │
                                           │            └──────────┘
                                           │
                                           └───────────>┌───────────┐
                                                        │ cancelled │
                                                        └───────────┘
```

| Status | Description |
|--------|-------------|
| `created` | Claim link initialized, no deposit yet |
| `depositing` | Deposit transaction submitted, waiting for confirmation |
| `deposited` | Tokens locked in escrow, link is claimable |
| `redeeming` | Claim transaction submitted, waiting for confirmation |
| `redeemed` | Receiver claimed the tokens |
| `refunding` | Refund transaction submitted |
| `refunded` | Sender reclaimed after expiration |
| `cancelled` | Sender cancelled before claim |
| `error` | Something went wrong |

Default expiration: 15 days from creation (configurable).

### Dashboard Links

**Create & Deposit (done by businesses via Dashboard web UI):**

1. Business connects wallet on the [Linkdrop Dashboard](https://dashboard.linkdrop.io).
2. Dashboard generates claim codes client-side in the browser, the codes are never sent to Linkdrop's server.
3. Business submits an approve transaction granting the Linkdrop escrow contract permission to transfer tokens on their behalf. Tokens stay in the campaign creator's wallet until each link is claimed.
4. Dashboard provides the claim links (e.g., as downloadable CSV) for the business to distribute via email, QR codes, etc.

The wallet does not need to implement any of the above. It only needs to support the redeem side.

**Redeem (handled by the wallet):**

1. SDK parses the claim URL. Dashboard links are detected automatically by the `src=d` query parameter.
2. SDK derives a `transferId` from the claim code (`ethers.id(claimCode)` → use as private key → public address).
3. SDK calls the Dashboard API to fetch link details (token, amount, chain, status, escrow address).
4. SDK generates a receiver signature from the claim code.
5. SDK calls the redeem endpoint with the receiver's wallet address and signature.
6. Linkdrop's relayer submits the on-chain redeem transaction. The receiver pays no gas.

The redeem flow is functionally the same as P2P — the SDK handles the API routing and URL parsing differences internally.

**Statuses:**

Dashboard links arrive already claimable — there is no deposit step to wait for. The SDK normalizes all statuses to the same values used by P2P links, so the wallet sees a consistent set of statuses regardless of link type: `deposited` (ready to claim), `redeeming`, `redeemed`, `cancelled` (expired or deactivated by the creator), `error`.

---

## Security & Privacy Model

### Anyone with the URL Can Claim
The claim URL contains a secret claim code (secp256k1 private key). There is no additional authentication — **possessing the URL is the only thing needed to redeem the tokens.** This is by design: it allows recipients to claim without having an existing wallet or account.

### Claim URL Never Stored Server-Side
The claim code is generated client-side and encoded into the URL. Linkdrop's backend never sees or stores the full claim URL. If the sender loses the URL, they can regenerate it using a wallet signature (`generateClaimUrl`).

### Escrow Contract Model (P2P)
Tokens are held in audited escrow contracts, not in Linkdrop-controlled wallets. Contract operations:
- **deposit** — sender locks tokens; only the sender's address is authorized
- **redeem** — requires a valid receiver signature derived from the claim code
- **refund** — tokens are reclaimed by Linkdrop relayer after the link is expired
- **cancel** — sender can cancel before the link is claimed

For Dashboard links, the model is different: tokens stay in the campaign creator's wallet (via an approve pattern) and are transferred directly to the receiver on claim.

### Relayer (Gas Sponsorship)
Linkdrop runs relayers that submit claim transactions on-chain and sponsor all gas fees. Recipients don't pay gas and don't need to have any crypto to claim a link. The relayer submits redeem and refund transactions, but the contract enforces all authorization checks — the relayer cannot steal funds, it can only relay valid signatures.

### Contracts

#### P2P Contracts

| Live since | Internally audited by | Source |
|------------|----------------------|--------|
| 2023 | Coinbase | [linkdrop-p2p-contracts](https://github.com/LinkdropHQ/linkdrop-p2p-contracts) |

Sender deposits tokens into the escrow contract. Tokens are held in the contract until claimed, refunded, or cancelled.

Deployed at the same addresses on all supported chains. "CBW" = Coinbase Wallet deployment; "Public" = available to all integrators.

| Address | Type | Deployment | Version |
|---------|------|------------|---------|
| `0x139b79602b68e8198ea3d57f5e6311fd98262269` | ERC20 + Native | Public | 3.1 |
| `0xe0cec4f0b66257fc6b13652c303237de0fd92ed8` | NFTs | Public | 3.1 |
| `0xedfea6336c922f896c7e09ba282beb0cb4476675` | ERC20 + Native | CBW | 3.1 |
| `0xff3471dfdc6f82694e5ad4d4e7ffedf23e1e38e0` | NFTs | CBW | 3.1 |
| `0xbe7b40eb3a9d85d3a76142cb637ab824f0d35ead` | ERC20 + Native | Public | 3.2 |
| `0x5fc1316119a1b7cec52a2984c62764343dca70c9` | NFTs | Public | 3.2 |
| `0x5badb0143f69015c5c86cbd9373474a9c8ab713b` | ERC20 + Native | CBW | 3.2 |
| `0x3c74782de03c0402d207fe41307fe50fe9b6b5c7` | NFTs | CBW | 3.2 |

For technical details, see the [P2P contracts README](https://github.com/LinkdropHQ/linkdrop-p2p-contracts/blob/main/README.md).

**Gasless Deposits (EIP-3009):**
For supported stablecoins, the sender signs an off-chain EIP-712 typed data authorization instead of submitting an on-chain transaction. The escrow contract calls `receiveWithAuthorization` or `approveWithAuthorization` on the token contract. No gas is required from the sender. Supported: USDC (all chains), EURC (Base), cbBTC (Base), bridged USDC (Polygon).

**Encrypted Sender Messages:**
Senders can attach a message (max 140 chars). The message is encrypted client-side using TweetNaCl secretbox (XSalsa20-Poly1305) with a key derived from a signature. The encryption key is embedded in the claim URL, so the receiver can decrypt it. Linkdrop's backend stores only the ciphertext.

#### Dashboard Contracts

| Live since | Internally audited by | Source |
|------------|----------------------|--------|
| 2019 | Ledger | [linkdrop-contracts](https://github.com/LinkdropHQ/linkdrop-contracts) |

Business approves the contract to transfer tokens on their behalf. Tokens stay in the business's wallet until each link is individually claimed.

Factory contract addresses (per chain):

| Chain | Chain ID | Address |
|-------|----------|---------|
| Ethereum | 1 | `0x50dADaF6739754fafE0874B906F60688dB483855` *(gas not sponsored)* |
| Polygon | 137 | `0x50dADaF6739754fafE0874B906F60688dB483855` |
| Base | 8453 | `0xa81880C06e925Ad1412773D621425796467A9C61` |

For a high-level technical description of how Dashboard links work, see the [Linkdrop Technical Description](https://blog.linkdrop.io/linkdrop-technical-description-2ec43f718924) blog post.

---

## API

- **API documentation:** [escrow-docs.linkdrop.io](https://escrow-docs.linkdrop.io)  
- **API URL:** `https://escrow-api.linkdrop.io`  
- **Authentication:** All requests require a Bearer token in the `Authorization` header. The API key is provided by Linkdrop.  


## Claim Link URL Format

All claim URLs use a hash-fragment structure: `{baseUrl}/#/code?{params}`. The `baseUrl` is configurable, wallets can use their own domain.

### P2P Links

```
{baseUrl}/#/code?k=GYJTMLAHEBdEWQPiF5wV4fbR84X9zT1hPs5dkShWDcv5&c=8453&v=3&src=p2p
```

| Parameter | Description |
|-----------|-------------|
| `k` | Claim code — Base58-encoded secp256k1 private key (32 bytes) |
| `c` | Chain ID |
| `v` | Link version |
| `src` | Link source — `p2p` |
| `m` | *(optional)* Message encryption key — present only if the sender attached an encrypted message |

The `transferId` is **not** included in standard P2P links. The SDK derives it from the claim code at decode time: `new ethers.Wallet(claimCode).address`.

**Recovered P2P link** (sender regenerated the URL via `generateClaimUrl`):

```
{baseUrl}/#/code?k=GbN1TPJMHPTkyFrQykuzAiiFtt8Vhnyrxa8QNs8SPZD8&sg=8HvhPbv3n7DsZSuyJFaNtzqdzEnmRK9x4h3BXNdmJdB9UHPYvqFamqWd2PUcVvNTmJCtgrziuMetnsJbM14rZboqL&i=3K9xVf2fb4kN8NUrQS7nuT3igeA4&c=8453&v=3&sgl=65&src=p2p
```

Additional parameters in recovered links:

| Parameter | Description |
|-----------|-------------|
| `sg` | Sender's EIP-712 signature (Base58-encoded) |
| `i` | Transfer ID (Base58-encoded, 20 bytes) |
| `sgl` | Signature byte length (always `65` for ECDSA) |

Recovered links are redeemed via the `/redeem-recovered` endpoint (includes sender signature verification), while standard links use `/redeem`.

### Dashboard Links

```
{baseUrl}/#/code?k=AdFBQPRHBLpW&c=8453&v=3&src=d
```

| Parameter | Description |
|-----------|-------------|
| `k` | Claim code |
| `c` | Chain ID |
| `v` | Link version |
| `src` | Link source — `d` for Dashboard |

The SDK detects the link type from the `src` parameter (`p2p` or `d`) and handles the differences automatically.

---

## SDK Documentation

For SDK initialization, usage examples, method reference, ClaimLink properties, versioning, and error codes, see the full SDK README:

**[Linkdrop SDK Documentation](https://github.com/LinkdropHQ/linkdrop-sdk/blob/main/README.md)**

---

## References

- **Integration Overview:** [wallet-integration-overview.md](wallet-integration-overview.md)
- **Linkdrop Technical Description (blog):** [blog.linkdrop.io/linkdrop-technical-description](https://blog.linkdrop.io/linkdrop-technical-description-2ec43f718924)
- **SDK repository & docs:** [github.com/LinkdropHQ/linkdrop-sdk](https://github.com/LinkdropHQ/linkdrop-sdk) · [README](https://github.com/LinkdropHQ/linkdrop-sdk/blob/main/README.md) · [npm](https://www.npmjs.com/package/linkdrop-sdk)
- **P2P contracts:** [github.com/LinkdropHQ/linkdrop-p2p-contracts](https://github.com/LinkdropHQ/linkdrop-p2p-contracts)
- **Dashboard contracts:** [github.com/LinkdropHQ/linkdrop-contracts](https://github.com/LinkdropHQ/linkdrop-contracts)
- **API documentation:** [escrow-docs.linkdrop.io](https://escrow-docs.linkdrop.io)

---

**Contact:** hi@linkdrop.io
