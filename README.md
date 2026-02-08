A Claude Code skill for executing Jupiter API operations on Solana. Includes scripts for fetching quotes, signing transactions, and executing swaps.

## Prerequisites

- Node.js 18+
- pnpm (recommended) or npm
- Jupiter API key from [portal.jup.ag](https://portal.jup.ag)
- Solana wallet (for signing transactions)

## Installation

```bash
# Clone the repo
cd /path/to/jup-skill

# Install dependencies
pnpm install
```

## Setup

### 1. Get API Key

1. Visit [portal.jup.ag](https://portal.jup.ag)
2. Create an account
3. Generate an API key

### 2. Configure Environment

```bash
# Set Jupiter API key
export JUP_API_KEY=your_api_key_here

# Optional: Set custom RPC (for Metis swaps)
export SOLANA_RPC_URL=https://your-rpc.com
```

### 3. Wallet Setup

You'll need a Solana wallet file to sign transactions.

> **SECURITY NOTE**: This tool only accepts wallet files, not private keys via command line arguments. This prevents exposure in shell history and process listings.

```bash
# Generate new wallet
solana-keygen new -o ~/.config/solana/id.json

# Or use existing wallet - the --wallet flag is required
pnpm wallet-sign -t "TX" --wallet ~/.config/solana/id.json

# Tilde expansion is supported
pnpm wallet-sign -t "TX" --wallet ~/my-wallets/trading.json
```

## Usage

### Quick Examples

```bash
# Search for tokens
pnpm fetch-api -e /ultra/v1/search -p '{"query":"SOL"}'

# Get swap order
pnpm fetch-api -e /ultra/v1/order -p '{
  "inputMint": "So11111111111111111111111111111111111111112",
  "outputMint": "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v",
  "amount": "1000000",
  "taker": "YOUR_WALLET_ADDRESS"
}'

# Sign a transaction (--wallet flag is required)
pnpm wallet-sign -t "BASE64_UNSIGNED_TX" --wallet ~/.config/solana/id.json

# Execute Ultra order
pnpm execute-ultra -r "REQUEST_ID" -t "BASE64_SIGNED_TX"

# Send transaction to RPC (for Metis)
# Note: Default public RPC is rate-limited. Use a dedicated RPC for production.
pnpm send-transaction -t "BASE64_SIGNED_TX"
```

### Complete Ultra Swap Flow

```bash
# 1. Get order
ORDER=$(pnpm fetch-api -e /ultra/v1/order -p '{
  "inputMint": "So11111111111111111111111111111111111111112",
  "outputMint": "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v",
  "amount": "1000000",
  "taker": "YOUR_WALLET_ADDRESS"
}')

# 2. Extract and sign (--wallet flag is required)
UNSIGNED_TX=$(echo "$ORDER" | jq -r '.transaction')
SIGNED_TX=$(pnpm wallet-sign -t "$UNSIGNED_TX" --wallet ~/.config/solana/id.json)

# 3. Execute
REQUEST_ID=$(echo "$ORDER" | jq -r '.requestId')
pnpm execute-ultra -r "$REQUEST_ID" -t "$SIGNED_TX"
```

## Scripts

| Script | Purpose |
|--------|---------|
| `fetch-api.ts` | Fetch from any Jupiter API endpoint |
| `wallet-sign.ts` | Sign transactions with local wallet file |
| `execute-ultra.ts` | Execute Ultra orders |
| `send-transaction.ts` | Send signed transactions to RPC |

## Claude Code Skill

This repo includes a `SKILL.md` file that can be used as a Claude Code skill. The skill provides:

- Quick reference for all scripts
- Complete workflows for Ultra and Metis swaps
- API endpoint reference
- Caveats and error handling guidance

### Installing as a Skill

Add to your Claude Code skills directory:

```bash
# Link or copy to skills directory
ln -s /path/to/jup-skill ~/.claude/skills/jup-skill
```

## Troubleshooting

### "API key required" error

Set the `JUP_API_KEY` environment variable or pass `--api-key` flag.

### "Rate limited (429)" error

- Free tier: 60 requests/minute
- Consider upgrading at [portal.jup.ag](https://portal.jup.ag)
- Ultra rate limits scale with swap volume

### "Transaction expired" error

Blockhash expired. Request a fresh order/quote and try again.

### "Insufficient funds" error

Check that your wallet has enough SOL for fees and the input token. Note: Even "gasless" Ultra swaps require SOL for account rent (creating token accounts).

## Resources

- [Jupiter Portal](https://portal.jup.ag) - API key management
- [Jupiter Docs](https://dev.jup.ag) - Full documentation
- [Status Page](https://status.jup.ag) - API status
- [Solscan](https://solscan.io) - Transaction explorer

## License

MIT
