# Billbee MCP Server

**English** · [Deutsch](docs/README.de.md)

**Connect Billbee to Claude, ChatGPT and Copilot: orders, products, customers and shipping providers as MCP tools.** Powered by [AnythingMCP](https://github.com/HelpCode-ai/anythingmcp).

Billbee MCP Server gives Claude, ChatGPT, Copilot and Cursor 8 tools for Billbee: orders, products, customers and shipping providers. Every tool only reads. It runs on AnythingMCP: one click on AnythingMCP Cloud, or self-hosted with Docker. Credentials are stored encrypted and every call is audited.

**Last verified:** 2026-09-26 against the Billbee REST API v1 (production traffic on AnythingMCP Cloud: 266 successful tool calls from 3 workspaces in the last 90 days).  
**Adapter synced:** <!-- synced -->2026-09-26

Maintained by [KOCH Freiburg GmbH](https://www.kochfreiburg.de/), which runs AnythingMCP in production. Built on [AnythingMCP](https://github.com/HelpCode-ai/anythingmcp) by helpcode.ai.

## Quick start (AnythingMCP Cloud)

1. Sign in at [cloud.anythingmcp.com](https://cloud.anythingmcp.com) and open the [install link](https://cloud.anythingmcp.com/connectors/store?install=billbee).
2. Enter `BILLBEE_API_KEY`, `BILLBEE_LOGIN_EMAIL`, `BILLBEE_API_PASSWORD` (see [Authentication](#authentication)).
3. Copy the URL of your MCP server under **MCP Servers** and add it to your AI client ([below](#connect-claude-chatgpt-copilot-or-cursor)).

AnythingMCP Cloud is the same open-source code, operated by helpcode.ai in Frankfurt, Germany.

## Self-hosted (Docker)

Needs Docker 24+, openssl and Node 18+.

```bash
git clone https://github.com/kochfreiburg/billbee-mcp-server.git
cd billbee-mcp-server
./scripts/install.sh
```

`install.sh` writes `.env` with fresh secrets, starts AnythingMCP, creates the first admin, installs the connector if `BILLBEE_API_KEY` and `BILLBEE_LOGIN_EMAIL` and `BILLBEE_API_PASSWORD` are set in `.env` and creates an MCP API key. Without credentials it prints the install link instead: `http://localhost:3000/connectors/store?install=billbee`. Then check the whole chain:

```bash
npm install && node scripts/smoke.mjs
```

## Connect Claude, ChatGPT, Copilot or Cursor

- **Claude (claude.ai, Desktop, mobile):** *Customize → Connectors → Add custom connector*, paste your MCP server URL and sign in. Claude connects from Anthropic's cloud, so the URL must be public HTTPS: your AnythingMCP Cloud URL, or your own instance behind TLS.
- **Claude Code:**

  ```bash
  claude mcp add --transport http billbee-mcp-server http://localhost:4000/mcp --header "X-API-Key: <MCP_API_KEY>"
  ```
- **Cursor** (`.cursor/mcp.json`) and **VS Code / GitHub Copilot** (`.vscode/mcp.json`, key `servers` instead of `mcpServers`, plus `"type": "http"`):

  ```json
  { "mcpServers": { "billbee-mcp-server": { "url": "http://localhost:4000/mcp", "headers": { "X-API-Key": "<MCP_API_KEY>" } } } }
  ```
- **ChatGPT:** add the public HTTPS URL as a connector (app) in ChatGPT's settings. A `localhost` URL does not work there.

## Tools

8 tools, generated from [`adapter/billbee.json`](adapter/billbee.json). **read** tools cannot change anything in the source system.

<!-- tools:start (generated from adapter/*.json, do not edit) -->
| Tool | What it does | Access |
|---|---|---|
| `billbee_list_orders` | List orders from Billbee with filtering by order date range, modification date (incremental sync) and pagination. | read |
| `billbee_get_order` | Get a single Billbee order by its internal Billbee order id, returning full detail including line items, addresses, payment and shipping info. | read |
| `billbee_get_order_by_extref` | Look up a Billbee order by its external reference (the marketplace/shop order number), returning the full order detail. | read |
| `billbee_list_products` | List products/articles from Billbee with pagination. | read |
| `billbee_get_product` | Get a single Billbee product by id (or by SKU/EAN via lookupBy), returning full article detail including pricing and stock. | read |
| `billbee_list_customers` | List customers from Billbee with pagination. | read |
| `billbee_get_customer_orders` | List all orders belonging to a specific Billbee customer by customer id, with pagination. | read |
| `billbee_list_shipping_providers` | List the shipping providers configured in the Billbee account, returning provider names and their available shipping products. | read |
<!-- tools:end -->

## Example prompts

- Which orders from yesterday are still unshipped?
- Find the Billbee order for Kaufland order number 123-456.
- Show all orders of customer 4711 this year.
- Which products are low on stock?
- What does the product with EAN 4006381333931 cost, and how many do we have?
- How many orders did we get per sales channel last week?

More in [examples/prompts.md](examples/prompts.md).

## Authentication

Billbee is the leading German/DACH multichannel order-management tool (20,000+ retailers) connecting marketplaces and shops (Amazon, eBay, Otto, Kaufland, Shopify, WooCommerce, etc.). This connector wraps the REST API at https://api.billbee.io/api/v1.

## Authentication (THREE credentials, all required)
Billbee requires both an application API key AND HTTP Basic Auth at the same time:
1. `BILLBEE_API_KEY` — sent in the `X-Billbee-Api-Key` header. Request it from Billbee (via their API request form / support@billbee.io, describing what you build).
2. `BILLBEE_LOGIN_EMAIL` — your Billbee account login email (Basic Auth username).
3. `BILLBEE_API_PASSWORD` — a dedicated API password (NOT your normal login password). Enable the API and set this password in the Billbee web app under Settings → Billbee API → General Settings.
All three are mandatory; the API key alone or Basic Auth alone will be rejected.

## Pagination
List endpoints use `page` (1-based) and `pageSize` (max 250). Responses wrap rows in a paged envelope: a `Data` array plus a `Paging` object with `Page`, `TotalPages`, `TotalRows`. Iterate `page` until `Page == TotalPages`.

## Orders
`billbee_list_orders` filters by `minOrderDate`/`maxOrderDate` and `modifiedAtMin`/`modifiedAtMax` (incremental sync). Order states are numeric; use `billbee_get_order` for full detail, or `billbee_get_order_by_extref` to look up by the marketplace's external order number.

## Rate limit
Hard limit of 2 requests per second per (API key + user); exceeding it returns HTTP 429 — back off and retry. The API has usage-based pricing above a free allotment, so avoid tight polling loops; prefer `modifiedAtMin`/`updatedOrNewSince`-style incremental queries.

## Security

- **Read or write is your choice.** All 8 tools only read. Assign the connector to an MCP server whose role whitelists only the tools you want, and the rest are invisible to that client.
- **Credentials** are encrypted with AES-256-GCM and never shown to the model.
- **Response mapping** drops or reshapes fields per tool before they reach the model, e.g. bank details or personal data.
- **Audit log:** every call is recorded with input, output, duration and status, in your own database when self-hosted.
- **SSO, RBAC and SCIM** are included in the self-hosted build.

## FAQ

### Is there a Billbee MCP server?
Yes, this one. It connects Billbee to Claude, ChatGPT and Copilot through AnythingMCP: 8 tools for orders, products, customers and shipping providers.

### What do I need to connect it?
Three values: an API key, which you request from Billbee; the e-mail you log in with; and a separate API password you set under **Settings → Billbee API** in Billbee. All three are required; the Authentication section has the steps.

### Can the AI change orders in Billbee?
No. All 8 tools only read.

### Can I look up an order by the marketplace's order number?
Yes. `billbee_get_order_by_extref` finds a Billbee order by the external reference from Amazon, eBay, Kaufland, OTTO or your shop.

### Does it work with ChatGPT and Copilot?
Yes. The same MCP server works in ChatGPT (with a public HTTPS URL such as AnythingMCP Cloud), GitHub Copilot in VS Code, Cursor and Claude Code.

## Troubleshooting

| Problem | Fix |
|---|---|
| `401` / `403` from the vendor | The credentials are wrong or lack rights. Re-enter them on the connector page; the import runs a test call and shows the result. |
| Tools missing in the AI client | The connector is not assigned to the MCP server the client uses. Check **MCP Servers**, then run `node scripts/smoke.mjs`. |
| The host is on your internal network | Self-host AnythingMCP on that network and add the hostname to `SSRF_ALLOWED_HOSTS`, or the outbound guard blocks the call. |
| Works locally, fails on AnythingMCP Cloud | The system must be reachable from the internet with a valid TLS certificate. |

## Related

- [ecommerce-mcp-server](https://github.com/HelpCode-ai/ecommerce-mcp-server): E-commerce MCP server: connect Amazon, eBay, WooCommerce, Shopware, Kaufland, OTTO and 7 more to Claude & ChatGPT.
- [AnythingMCP](https://github.com/HelpCode-ai/anythingmcp): the open-source MCP server and gateway this repository is built on.

## License

AGPL-3.0-only. The adapter definition in `adapter/` comes from AnythingMCP (AGPL-3.0).
