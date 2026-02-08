---
name: jupiter-skill
description: Execute Jupiter API operations on Solana - fetch quotes, sign transactions, execute swaps. Use when implementing token swaps, DCA, limit orders, or any Jupiter integration. Includes scripts for Ultra and Metis swap flows.
---

# Jupiter API Skill

Execute Jupiter API operations through 4 utility scripts for fetching data, signing transactions, and executing swaps on Solana.

**Base URL**: `https://api.jup.ag`

## Quick Reference

| Task | Script | Example |
|------|--------|---------|
| Fetch any Jupiter API | `fetch-api.ts` | `pnpm fetch-api -e /ultra/v1/search -p '{"query":"SOL"}'` |
| Sign a transaction | `wallet-sign.ts` | `pnpm wallet-sign -t "BASE64_TX" -w ~/.config/solana/id.json` |
| Execute Ultra order | `execute-ultra.ts` | `pnpm execute-ultra -r "REQUEST_ID" -t "SIGNED_TX"` |
| Send tx to RPC | `send-transaction.ts` | `pnpm send-transaction -t "SIGNED_TX"` |

## Setup

Install dependencies before using scripts:
```bash
cd /path/to/jup-skill
pnpm install
```

## API Key Setup

**ALWAYS required.** All Jupiter API endpoints require an `x-api-key` header.

1. Visit [portal.jup.ag](https://portal.jup.ag)
2. Create account and generate API key
3. Set via environment variable (recommended):
   ```bash
   export JUP_API_KEY=your_api_key_here
   ```
   Or pass via `--api-key` flag on each command.

## Scripts

### fetch-api.ts

Fetch data from any Jupiter API endpoint.

```bash
# Search for tokens
pnpm fetch-api -e /ultra/v1/search -p '{"query":"SOL"}'

# Get Ultra swap order (quote + unsigned transaction)
pnpm fetch-api -e /ultra/v1/order -p '{
  "inputMint": "So11111111111111111111111111111111111111112",
  "outputMint": "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v",
  "amount": "1000000",
  "taker": "YOUR_WALLET_ADDRESS"
}'

# Get Metis quote
pnpm fetch-api -e /swap/v1/quote -p '{
  "inputMint": "So11111111111111111111111111111111111111112",
  "outputMint": "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v",
  "amount": "1000000",
  "slippageBps": "50"
}'

# POST request (for Metis swap transaction)
pnpm fetch-api -e /swap/v1/swap -m POST -b '{
  "quoteResponse": {...},
  "userPublicKey": "YOUR_WALLET"
}'
```

**Arguments:**
- `-e, --endpoint` (required): API path, e.g., `/ultra/v1/order`
- `-p, --params`: Query params (GET) or body (POST) as JSON string
- `-b, --body`: Request body for POST requests
- `-m, --method`: HTTP method, `GET` (default) or `POST`
- `-k, --api-key`: API key (or use `JUP_API_KEY` env var)

### wallet-sign.ts

Sign transactions using a local wallet file.

> **SECURITY NOTE**: The `--wallet` flag is required. This script does not accept private keys via command line arguments to prevent exposure in shell history and process listings.

```bash
# Using Solana CLI wallet (JSON array format)
pnpm wallet-sign -t "BASE64_UNSIGNED_TX" --wallet ~/.config/solana/id.json

# Tilde expansion is supported
pnpm wallet-sign -t "BASE64_UNSIGNED_TX" --wallet ~/my-wallets/trading.json
```

**Arguments:**
- `-t, --unsigned-tx` (required): Base64-encoded unsigned transaction
- `-w, --wallet` (required): Path to Solana CLI JSON wallet file (supports ~ for home directory)

**Output:** Signed transaction (base64) to stdout.

### execute-ultra.ts

Execute Ultra orders after signing.

```bash
pnpm execute-ultra -r "REQUEST_ID_FROM_ORDER" -t "BASE64_SIGNED_TX"
```

**Arguments:**
- `-r, --request-id` (required): Request ID from `/ultra/v1/order` response
- `-t, --signed-tx` (required): Base64-encoded signed transaction
- `-k, --api-key`: API key (or use `JUP_API_KEY` env var)

**Output:** Execution result JSON including signature and status.

### send-transaction.ts

Send signed transactions to Solana RPC. **Use for Metis swaps** (Ultra handles RPC internally).

> **Warning**: The default public Solana RPC (`api.mainnet-beta.solana.com`) is rate-limited and unreliable for production use. Use a dedicated RPC provider (Helius, QuickNode, Triton, etc.) for production applications.

```bash
# Default RPC (mainnet-beta)
pnpm send-transaction -t "BASE64_SIGNED_TX"

# Custom RPC
pnpm send-transaction -t "BASE64_SIGNED_TX" -r "https://your-rpc.com"

# With environment variable
export SOLANA_RPC_URL="https://your-rpc.com"
pnpm send-transaction -t "BASE64_SIGNED_TX"
```

**Arguments:**
- `-t, --signed-tx` (required): Base64-encoded signed transaction
- `-r, --rpc-url`: RPC endpoint (default: `https://api.mainnet-beta.solana.com`)
- `--skip-preflight`: Skip preflight checks (faster, less safe)
- `--max-retries`: Max send retries (default: 3)

**Output:** Transaction signature to stdout.

---

## Workflows

### Ultra Swap (Recommended)

Ultra is RPC-less, gasless, with automatic slippage optimization.

```
Ultra Swap Progress:
- [ ] Step 1: Get order from /ultra/v1/order
- [ ] Step 2: Sign the transaction
- [ ] Step 3: Execute via /ultra/v1/execute
- [ ] Step 4: Verify result
```

**Step 1: Get Order**

```bash
ORDER=$(pnpm fetch-api -e /ultra/v1/order -p '{
  "inputMint": "So11111111111111111111111111111111111111112",
  "outputMint": "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v",
  "amount": "1000000",
  "taker": "YOUR_WALLET_ADDRESS"
}')
echo "$ORDER"
```

Response contains `requestId` and `transaction` (base64 unsigned).

**Step 2: Sign Transaction**

Extract transaction from response and sign:

```bash
UNSIGNED_TX=$(echo "$ORDER" | jq -r '.transaction')
SIGNED_TX=$(pnpm wallet-sign -t "$UNSIGNED_TX" -w ~/.config/solana/id.json)
```

**Step 3: Execute Order**

```bash
REQUEST_ID=$(echo "$ORDER" | jq -r '.requestId')
pnpm execute-ultra -r "$REQUEST_ID" -t "$SIGNED_TX"
```

**Step 4: Verify**

Check the signature on [Solscan](https://solscan.io).

---

### Metis Swap (Advanced)

Use Metis when you need custom transaction composition or fine-grained control.

```
Metis Swap Progress:
- [ ] Step 1: Get quote from /swap/v1/quote
- [ ] Step 2: Build transaction via /swap/v1/swap
- [ ] Step 3: Sign the transaction
- [ ] Step 4: Send to RPC
- [ ] Step 5: Verify on-chain
```

**Step 1: Get Quote**

```bash
QUOTE=$(pnpm fetch-api -e /swap/v1/quote -p '{
  "inputMint": "So11111111111111111111111111111111111111112",
  "outputMint": "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v",
  "amount": "1000000",
  "slippageBps": "50"
}')
```

**Step 2: Build Transaction**

```bash
SWAP_TX=$(pnpm fetch-api -e /swap/v1/swap -m POST -b "{
  \"quoteResponse\": $QUOTE,
  \"userPublicKey\": \"YOUR_WALLET_ADDRESS\",
  \"dynamicComputeUnitLimit\": true,
  \"prioritizationFeeLamports\": \"auto\"
}")
```

**Step 3: Sign**

```bash
UNSIGNED_TX=$(echo "$SWAP_TX" | jq -r '.swapTransaction')
SIGNED_TX=$(pnpm wallet-sign -t "$UNSIGNED_TX" --wallet ~/.config/solana/id.json)
```

**Step 4: Send to RPC**

```bash
pnpm send-transaction -t "$SIGNED_TX" -r "https://your-rpc.com"
```

**Step 5: Verify**

Check signature on Solscan.

---

## API Endpoints Reference

| Use Case | API | Endpoint |
|----------|-----|----------|
| Token swaps (default) | Ultra | `/ultra/v1/order`, `/ultra/v1/execute` |
| Swaps with control | Metis | `/swap/v1/quote`, `/swap/v1/swap` |
| Limit orders | Trigger | `/trigger/v1/createOrder`, `/trigger/v1/execute` |
| DCA | Recurring | `/recurring/v1/createOrder`, `/recurring/v1/execute` |
| Token search | Ultra | `/ultra/v1/search` |
| Token holdings | Ultra | `/ultra/v1/holdings/{address}` |
| Token warnings | Ultra | `/ultra/v1/shield` |
| Token prices | Price | `/price/v3?ids={mints}` |
| Token metadata | Tokens | `/tokens/v2/search?query={query}` |
| Portfolio | Portfolio | `/portfolio/v1/positions?wallet={address}` |

---

## Caveats & Limitations

### API Key & Rate Limits

| Tier | Rate Limit |
|------|------------|
| Free | 60 requests/minute |
| Pro | Up to 30,000 requests/minute |
| Ultra | Dynamic scaling with executed swap volume |

Ultra rate limits increase as you execute more swaps. Base: 50 requests per 10-second window.

### Fees

| API | Fee |
|-----|-----|
| Ultra | 5-10 basis points per swap |
| Metis | No Jupiter fee (you pay gas) |

Integrators can add custom fees (50-255 bps). Jupiter takes 20% of integrator fees.

### Gasless Requirements

Ultra offers "gasless" swaps where Jupiter pays the transaction fees, but with important caveats:
- **User still needs SOL** for account rent (creating token accounts)
- **User must sign** the transaction (not truly "zero-touch")
- **Minimum trade amount**: ~$10 equivalent
- Automatic when taker has <0.01 SOL
- JupiterZ RFQ: market makers pay transaction fees

### Transaction Size Limit

Solana transactions are limited to **1232 bytes**. If you hit this:
- Reduce `maxAccounts` parameter in quote request
- Use `dynamicComputeUnitLimit: true` for Metis

### Token Limitations

- **Token2022**: NOT supported for Recurring (DCA) orders
- Some tokens may have transfer fees or freeze authority

### Ultra vs Metis

| Feature | Ultra | Metis |
|---------|-------|-------|
| RPC required | No (Jupiter handles) | Yes (your RPC) |
| Gasless | Yes (conditions apply) | No |
| Custom instructions | No | Yes |
| Transaction composition | No | Yes |
| Slippage | Auto-optimized | Manual |

**Use Ultra** for most swaps. **Use Metis** only when you need to add custom instructions or compose transactions.

### Common Errors

| Error | Cause | Fix |
|-------|-------|-----|
| `TransactionExpiredBlockhashNotFound` | Blockhash expired | Request fresh order/quote |
| `SlippageToleranceExceeded` | Price moved too much | Increase slippage or retry |
| `InsufficientFunds` | Not enough SOL/tokens | Check balances |
| `RateLimited (429)` | Too many requests | Wait and retry, or upgrade tier |
| `InvalidSignature` | Wrong signer or corrupted tx | Verify wallet matches taker address |

### Ultra Execute Error Codes

| Code | Meaning |
|------|---------|
| 0 | Success |
| -1 to -5 | Client/validation errors |
| -1000 to -1999 | Aggregator routing errors |
| -2000 to -2999 | RFQ (market maker) errors |

---

## Environment Variables

| Variable | Description | Default |
|----------|-------------|---------|
| `JUP_API_KEY` | Jupiter API key | (required) |
| `SOLANA_RPC_URL` | RPC endpoint for send-transaction | `https://api.mainnet-beta.solana.com` |

---

## Common Token Mints

| Token | Mint Address |
|-------|--------------|
| SOL (wrapped) | `So11111111111111111111111111111111111111112` |
| USDC | `EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v` |
| USDT | `Es9vMFrzaCERmJfrF4H2FYD4KCoNkY11McCe8BenwNYB` |
| JUP | `JUPyiwrYJFskUPiHa7hkeR8VUtAeFoSYbKedZNsDvCN` |

---

## Resources

- [Jupiter Portal](https://portal.jup.ag) - API key management
- [Jupiter Docs](https://dev.jup.ag) - Full documentation
- [Status Page](https://status.jup.ag) - API status
- [Solscan](https://solscan.io) - Transaction explorer
