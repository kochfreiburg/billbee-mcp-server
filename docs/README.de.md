# Billbee MCP Server

[English](../README.md) · **Deutsch**

**Verbinde Billbee mit Claude, ChatGPT und Copilot: orders, products, customers and shipping providers als MCP-Tools.** Basiert auf [AnythingMCP](https://github.com/HelpCode-ai/anythingmcp).

Billbee MCP Server gibt Claude, ChatGPT, Copilot und Cursor 8 Tools für Billbee: orders, products, customers and shipping providers. Alle Tools lesen nur. Es läuft auf AnythingMCP: mit einem Klick in AnythingMCP Cloud oder selbst gehostet mit Docker. Zugangsdaten werden verschlüsselt gespeichert, jeder Aufruf landet im Audit-Log.

**Zuletzt geprüft:** 2026-09-26 gegen the Billbee REST API v1 (production traffic on AnythingMCP Cloud: 266 successful tool calls from 3 workspaces in the last 90 days).  
**Adapter synchronisiert:** <!-- synced -->2026-09-26

Maintained by [KOCH Freiburg GmbH](https://www.kochfreiburg.de/), which runs AnythingMCP in production. Built on [AnythingMCP](https://github.com/HelpCode-ai/anythingmcp) by helpcode.ai.

## Schnellstart (AnythingMCP Cloud)

1. Melde dich bei [cloud.anythingmcp.com](https://cloud.anythingmcp.com) an und öffne den [Installationslink](https://cloud.anythingmcp.com/connectors/store?install=billbee).
2. Trage `BILLBEE_API_KEY`, `BILLBEE_LOGIN_EMAIL`, `BILLBEE_API_PASSWORD` ein (siehe [Authentifizierung](#authentifizierung)).
3. Kopiere die URL deines MCP-Servers unter **MCP Servers** und füge sie in deinen KI-Client ein ([siehe unten](#claude-chatgpt-copilot-oder-cursor-verbinden)).

AnythingMCP Cloud ist derselbe Open-Source-Code, betrieben von helpcode.ai in Frankfurt.

## Selbst gehostet (Docker)

Benötigt Docker 24+, openssl und Node 18+.

```bash
git clone https://github.com/kochfreiburg/billbee-mcp-server.git
cd billbee-mcp-server
./scripts/install.sh
```

`install.sh` schreibt `.env` mit neuen Secrets, startet AnythingMCP, legt den ersten Admin an, installiert den Connector, sofern `BILLBEE_API_KEY` und `BILLBEE_LOGIN_EMAIL` und `BILLBEE_API_PASSWORD` in `.env` gesetzt sind, und erzeugt einen MCP-API-Key. Ohne Zugangsdaten gibt es stattdessen den Installationslink aus: `http://localhost:3000/connectors/store?install=billbee`. Danach die ganze Kette prüfen:

```bash
npm install && node scripts/smoke.mjs
```

## Claude, ChatGPT, Copilot oder Cursor verbinden

- **Claude (claude.ai, Desktop, Mobil):** *Customize → Connectors → Add custom connector*, MCP-Server-URL einfügen und anmelden. Claude verbindet sich aus der Cloud von Anthropic, die URL muss also öffentlich per HTTPS erreichbar sein: deine AnythingMCP-Cloud-URL oder deine eigene Instanz mit TLS.
- **Claude Code:**

  ```bash
  claude mcp add --transport http billbee-mcp-server http://localhost:4000/mcp --header "X-API-Key: <MCP_API_KEY>"
  ```
- **Cursor** (`.cursor/mcp.json`) und **VS Code / GitHub Copilot** (`.vscode/mcp.json`, Schlüssel `servers` statt `mcpServers`, dazu `"type": "http"`):

  ```json
  { "mcpServers": { "billbee-mcp-server": { "url": "http://localhost:4000/mcp", "headers": { "X-API-Key": "<MCP_API_KEY>" } } } }
  ```
- **ChatGPT:** die öffentliche HTTPS-URL in den ChatGPT-Einstellungen als Connector (App) hinzufügen. Eine `localhost`-URL funktioniert dort nicht.

## Tools

8 Tools, erzeugt aus [`adapter/billbee.json`](../adapter/billbee.json). Tools mit **lesen** können im Quellsystem nichts ändern.

<!-- tools:start (generated from adapter/*.json, do not edit) -->
| Tool | Funktion | Zugriff |
|---|---|---|
| `billbee_list_orders` | List orders from Billbee with filtering by order date range, modification date (incremental sync) and pagination. | lesen |
| `billbee_get_order` | Get a single Billbee order by its internal Billbee order id, returning full detail including line items, addresses, payment and shipping info. | lesen |
| `billbee_get_order_by_extref` | Look up a Billbee order by its external reference (the marketplace/shop order number), returning the full order detail. | lesen |
| `billbee_list_products` | List products/articles from Billbee with pagination. | lesen |
| `billbee_get_product` | Get a single Billbee product by id (or by SKU/EAN via lookupBy), returning full article detail including pricing and stock. | lesen |
| `billbee_list_customers` | List customers from Billbee with pagination. | lesen |
| `billbee_get_customer_orders` | List all orders belonging to a specific Billbee customer by customer id, with pagination. | lesen |
| `billbee_list_shipping_providers` | List the shipping providers configured in the Billbee account, returning provider names and their available shipping products. | lesen |
<!-- tools:end -->

## Beispiel-Prompts

- Welche Bestellungen von gestern sind noch nicht versendet?
- Finde die Billbee-Bestellung zur Kaufland-Bestellnummer 123-456.
- Zeig mir alle Bestellungen von Kunde 4711 in diesem Jahr.
- Welche Artikel haben einen niedrigen Bestand?
- Was kostet der Artikel mit der EAN 4006381333931, und wie viele haben wir?
- Welche Versandanbieter sind eingerichtet?

Weitere (auf Englisch) in [examples/prompts.md](../examples/prompts.md).

## Authentifizierung

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

## Sicherheit

- **Lesen oder schreiben entscheidest du.** Alle 8 Tools lesen nur. Weise den Connector einem MCP-Server zu, dessen Rolle nur die gewünschten Tools freigibt; die anderen sieht dieser Client gar nicht.
- **Zugangsdaten** werden mit AES-256-GCM verschlüsselt und nie an das Modell gegeben.
- **Response-Mapping** entfernt oder formt Felder pro Tool, bevor sie das Modell erreichen, etwa Bankdaten oder personenbezogene Daten.
- **Audit-Log:** Jeder Aufruf wird mit Eingabe, Ausgabe, Dauer und Status protokolliert, selbst gehostet in deiner eigenen Datenbank.
- **SSO, RBAC und SCIM** sind in der selbst gehosteten Version enthalten.

## FAQ

### Gibt es einen MCP-Server für Billbee?
Ja, diesen hier. Er verbindet Billbee über AnythingMCP mit Claude, ChatGPT und Copilot: 8 Tools für Bestellungen, Artikel, Kunden und Versandanbieter.

### Was brauche ich für die Verbindung?
Drei Werte: einen API-Key, den du bei Billbee beantragst, die E-Mail-Adresse deines Logins und ein eigenes API-Passwort, das du in Billbee unter **Einstellungen → Billbee API** festlegst. Alle drei sind nötig.

### Kann die KI Bestellungen in Billbee ändern?
Nein. Alle 8 Tools lesen nur.

### Kann ich eine Bestellung über die Marktplatz-Bestellnummer finden?
Ja. `billbee_get_order_by_extref` findet eine Billbee-Bestellung über die externe Referenz von Amazon, eBay, Kaufland, OTTO oder deinem Shop.

## Fehlerbehebung

| Problem | Lösung |
|---|---|
| `401` / `403` vom Hersteller | Zugangsdaten falsch oder ohne Rechte. Auf der Connector-Seite neu eintragen; der Import macht einen Testaufruf und zeigt das Ergebnis. |
| Tools fehlen im KI-Client | Der Connector ist nicht dem MCP-Server zugewiesen, den der Client nutzt. **MCP Servers** prüfen, dann `node scripts/smoke.mjs` ausführen. |
| Das System steht im internen Netz | AnythingMCP in diesem Netz selbst hosten und den Hostnamen in `SSRF_ALLOWED_HOSTS` eintragen, sonst blockiert der Outbound-Guard den Aufruf. |
| Lokal ok, in AnythingMCP Cloud nicht | Das System muss aus dem Internet mit gültigem TLS-Zertifikat erreichbar sein. |

## Verwandte Repositories

- [ecommerce-mcp-server](https://github.com/HelpCode-ai/ecommerce-mcp-server): E-commerce MCP server: connect Amazon, eBay, WooCommerce, Shopware, Kaufland, OTTO and 7 more to Claude & ChatGPT.
- [AnythingMCP](https://github.com/HelpCode-ai/anythingmcp): der Open-Source-MCP-Server und -Gateway, auf dem dieses Repository aufbaut.

## Lizenz

AGPL-3.0-only. Die Adapter-Definition in `adapter/` stammt aus AnythingMCP (AGPL-3.0).
