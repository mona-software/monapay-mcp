# monapay-mcp

An MCP server that lets AI coding agents such as Claude Code, Cursor and Codex (or any MCP client) work with MONA Pay while coding: link a bank account, create checkout links and VietQR codes, look up incoming transfers, configure and test webhooks, email and Zalo group notifications, and verify webhook signatures.

[![test](https://github.com/mona-software/monapay-mcp/actions/workflows/test.yml/badge.svg)](https://github.com/mona-software/monapay-mcp/actions/workflows/test.yml)

Published on npm as `monapay-mcp` and in the MCP Registry as `vn.monapay/monapay-mcp`.

## Install

Requires Node.js 18 or later and a MONA Pay API key (client ID and client secret, created in the MONA Pay dashboard).

```bash
npx -y monapay-mcp   # starts the server over stdio
```

### Claude Code

```bash
claude mcp add monapay \
  -e MONAPAY_CLIENT_ID=your_client_id \
  -e MONAPAY_CLIENT_SECRET=your_client_secret \
  -- npx -y monapay-mcp
```

### Cursor (`.cursor/mcp.json`) and Claude Desktop (`claude_desktop_config.json`)

```json
{
  "mcpServers": {
    "monapay": {
      "command": "npx",
      "args": ["-y", "monapay-mcp"],
      "env": { "MONAPAY_CLIENT_ID": "client_id", "MONAPAY_CLIENT_SECRET": "client_secret" }
    }
  }
}
```

### Codex (`~/.codex/config.toml`)

```toml
[mcp_servers.monapay]
command = "npx"
args = ["-y", "monapay-mcp"]
env = { MONAPAY_CLIENT_ID = "client_id", MONAPAY_CLIENT_SECRET = "client_secret" }
```

## Configuration

| Variable | Required | Meaning |
| --- | --- | --- |
| `MONAPAY_CLIENT_ID` | yes (recommended) | `client_id` of an API key created in the dashboard |
| `MONAPAY_CLIENT_SECRET` | yes (recommended) | `client_secret`, shown once; used to obtain the OAuth token and authorize write actions |
| `MONAPAY_USERNAME` | legacy | MONA Pay username, used only when client credentials are not set |
| `MONAPAY_PASSWORD` | legacy | Password; does not work for accounts with 2FA enabled |
| `MONAPAY_BASE_URL` | no | Defaults to `https://api.monapay.vn` |

Client credentials are recommended over username/password, because accounts with 2FA enabled cannot log in with a password.

## Usage

### Tools

| Tool | What it does |
| --- | --- |
| `monapay_quickstart` | Entry point: checks account state (bank/VA linked, webhook configured) and returns the next steps |
| `monapay_whoami` | Verifies the connection and returns the account name and plan |
| `monapay_me` | Profile of the logged-in account |
| `monapay_link_bank_start` · `monapay_link_bank_verify_otp` | Link an ACB account and create a virtual account (first OTP) |
| `monapay_notification_register` · `monapay_notification_verify_otp` | Turn on incoming-transfer notifications (second OTP) |
| `monapay_list_bank_accounts` · `monapay_list_virtual_accounts` | Linked bank accounts and their virtual accounts (VA) |
| `monapay_get_payment_profile` · `monapay_set_payment_profile` | View or set the profile used by the hosted checkout page |
| `monapay_create_checkout` · `monapay_get_checkout` · `monapay_list_checkouts` · `monapay_cancel_checkout` | Create, inspect, filter and cancel hosted checkout sessions |
| `monapay_create_qr` · `monapay_cancel_qr` | Create or cancel a dynamic VietQR for an order |
| `monapay_list_transactions` | Incoming transactions for a VA (up to 100 per page), for reconciliation |
| `monapay_get_transaction` | Find one incoming transaction by `transaction_code` or ID within a VA |
| `monapay_get_transactions_summary` | Incoming totals, transaction count and daily data for a date range |
| `monapay_sandbox_transaction` | Create a fake incoming transaction; webhooks, Telegram, email and Zalo fire as for a real one. Without a VA, a sandbox VA is issued |
| `monapay_list_webhooks` · `monapay_create_webhook` · `monapay_update_webhook` · `monapay_delete_webhook` | Webhook configs (`NONE`, `API_KEY` or `HMAC_SHA256`; HMAC recommended) |
| `monapay_test_webhook` · `monapay_webhook_logs` · `monapay_webhook_stats` | Send a dummy webhook, per-delivery history, success rate / P95 |
| `monapay_list_email_configs` · `monapay_create_email_config` · `monapay_update_email_config` · `monapay_delete_email_config` | Email notification configs with 1 to 10 recipients |
| `monapay_verify_email` · `monapay_resend_email_verification` · `monapay_test_email` | Verify a recipient with a 6-digit code, resend the code, send a test email |
| `monapay_email_logs` · `monapay_email_stats` | Delivery metadata, success rate / P95 and error groups |
| `monapay_list_email_suppressions` · `monapay_remove_email_suppression` | View and remove suppressed addresses |
| `monapay_list_zalo_groups` · `monapay_create_zalo_group` · `monapay_update_zalo_group` · `monapay_delete_zalo_group` | Zalo group notification configs |
| `monapay_test_zalo_group` · `monapay_zalo_group_logs` | Send a test message and read Zalo group delivery history |
| `monapay_retry_transaction` | Re-send the webhook or Telegram notification for a transaction |
| `monapay_generate_key` | Generate a new `client_secret` |
| `monapay_rotate_key` | Rotate the secret of the current key when it may be exposed |
| `monapay_verify_signature` | Verify a webhook signature locally, with no network call |
| `monapay_generate_webhook_snippet` | Webhook receiver code for PHP, Node or Python, with a sample payload |

### Resources and prompt

- `monapay://docs/llms`: machine-readable index of the MONA Pay docs.
- `monapay://docs/{slug}`: one docs page as Markdown, for example `monapay://docs/webhooks/tich-hop-webhook`.
- Prompt `integrate-monapay` (arguments `language`, `framework`): a 6-step plan for the agent to integrate MONA Pay into the current project.

### Example

> "Integrate automatic bank transfer confirmation into this Laravel site with MONA Pay."

The agent calls `monapay_whoami`, gets receiver code with `monapay_generate_webhook_snippet(language="php")`, writes the endpoint into the project, registers it with `monapay_create_webhook(auth_type="HMAC_SHA256", secret_key=...)`, fires `monapay_test_webhook`, reads `monapay_webhook_logs` to confirm the endpoint returns 200, then uses `monapay_create_checkout` for each order and waits for `CHECKOUT_PAID`.

### Link a bank account with OTP

ACB sends each OTP to the account holder's registered phone. The agent must ask the user at steps 2 and 4 and never guess an OTP.

1. Call `monapay_link_bank_start` with the ACB account number, phone number, customer type, VA prefix and identifier. It returns `acb_request_id`, and ACB sends the first OTP.
2. Ask the user for the OTP and call `monapay_link_bank_verify_otp`. It returns `virtual_account_id` and the VA number.
3. Call `monapay_notification_register` with `virtual_account_id`. ACB sends the second OTP.
4. Ask the user for the second OTP and call `monapay_notification_verify_otp`. Incoming transfers are now forwarded to your configured webhooks.

If an OTP is wrong or expired, call the previous step again to get a new code.

### Zalo group notifications

The group must contain the MONA bot (Gấu Mona), which is added by the MONA team. `group_id` is 10 to 25 digits and comes from MONA Account/PMS.

Call `monapay_create_zalo_group` with `group_id`, a friendly name and the events to receive: `TRANSACTION_IN`, `CHECKOUT_PAID`, `WEBHOOK_FAILED`, `VA_CREATED`. You can limit it to one `virtual_account_id`, send a test with `monapay_test_zalo_group` and check delivery with `monapay_zalo_group_logs`.

Zalo does not render Markdown, so `message_template` must be plain text. It supports `{amount}`, `{description}`, `{virtual_account_number}`, `{transaction_code}` and `{transfer_date}`; the `{{amount}}` form also works.

### Key rotation

`monapay_rotate_key` uses the current secret to rotate the key that matches `MONAPAY_CLIENT_ID`. The old secret stops working immediately; update `MONAPAY_CLIENT_SECRET` in your MCP config and restart the client.

### Webhook signature

`X-Mona-Signature: sha256=<hex>`, where `hex = HMAC-SHA256(secret, "<X-Mona-Timestamp>.<raw_body>")`. Reject requests whose timestamp is more than 300 seconds off and compare in constant time. `transaction_code` stays the same across retries, so use it as the deduplication key.

## Development

```bash
npm install
npm test     # build + node --test

# Calls a few read-only tools against a real account
MONAPAY_CLIENT_ID=... MONAPAY_CLIENT_SECRET=... MONAPAY_BASE_URL=https://api.monapay.vn npm run smoke
```

Agent guide: https://monapay.vn/ai-agent · Documentation: https://monapay.vn/docs

## License

MIT

**MONA Pay is part of MONA Cloud by The MONA Group.**
