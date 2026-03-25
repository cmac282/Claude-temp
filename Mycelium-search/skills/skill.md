---
name: fablefood-context
description: "Knowledge map for Fable Food: tells Claude where to find information across all connected tools and systems. Load this at the start of any research or data task."
---

# Fable Food — Information Context Schema

This document tells Claude exactly where to look when answering questions for the Fable Food team. Before starting any multi-step research or data task, refer to the relevant section below to identify the right tool and location.

---

## 1. Financial Data → **BigQuery (Xero)**

**Best first stop for any financial question (revenue, costs, P&L, balance sheet, budget).**

| What you need | Where to look |
|---|---|
| Monthly P&L, revenue, cost of sales by region | `xero_processed_data.v_pnl_from_reports` — use `group`, `line`, `entity` (AU/UK/US), `period` columns |
| Balance sheet | `xero_processed_data.v_balance_sheet_from_reports` |
| Budget vs actuals / variance | `xero_processed_data.v_budget_variance` or `v_budget_summary_from_reports` |
| Raw invoices, bank transactions, contacts | `xero_datasources.invoices_au/uk/us`, `bank_transactions_au`, `contacts_au/uk/us` |
| Historical P&L by year | `xero_datasources.pnl_YYYY_au/uk/us` (available from 2019) |

**Key schema notes:**
- `entity` = `AU`, `UK`, or `US`
- `group` values: `Income` (AU/UK) or `Revenue` (US) for sales lines; `Cost of Sales`; `Operating Expenses`; `Gross Profit`; `Net Profit`
- Sales lines: `line = 'Sales'` and `line = 'Sales - eCommerce'`
- Amounts available in: `amountInCurrency` (local), `amountAUD`, `amountUSD`, `amountGBP`
- AU entity: `amountUSD` may be 0 — use `amountAUD` and convert if needed
- `period` is a DATE (first day of each month)

**Example query pattern:**
```sql
SELECT entity, period, SUM(amountInCurrency) AS sales, currency
FROM `xero_processed_data.v_pnl_from_reports`
WHERE group IN ('Income', 'Revenue') AND line = 'Sales'
  AND period >= DATE_TRUNC(DATE_SUB(CURRENT_DATE(), INTERVAL 6 MONTH), MONTH)
GROUP BY entity, period, currency
ORDER BY period, entity
```

---

## 2. Inventory & Orders → **BigQuery (Fishbowl)**

**For stock levels, sales orders, product/part data, and vendor info.**

| What you need | Where to look |
|---|---|
| Sales orders | `fishbowl_datasources.so` / `so_uk` / `so_us` + `soitem` / `soitem_uk` / `soitem_us` |
| Current inventory levels | `fishbowl_datasources.qtyinventory` / `qtyinventory_uk` / `qtyinventory_us` |
| Product / SKU list | `fishbowl_datasources.product` / `product_uk` / `product_us` |
| Part details & costs | `fishbowl_datasources.part` / `partcost` / `partcosthistory` |
| Customer list | `fishbowl_datasources.customer` / `customer_uk` / `customer_us` |
| Vendors | `fishbowl_datasources.vendor` / `vendorparts` |
| US open inventory weekly snapshot | `fishbowl_reporting.weekly_open_inv_us` |

**Note:** Tables follow the pattern `tablename`, `tablename_uk`, `tablename_us` for each region.

---

## 3. CRM / Sales Pipeline → **HubSpot** or **BigQuery (HubSpot)**

**For contacts, companies, deals, pipeline stage, and deal values.**

- **HubSpot MCP tools** (`search_crm_objects`, `get_crm_objects`): use for live lookups of deals, contacts, and companies
- **BigQuery** `hubspot_datasources`: tables `c_companies`, `c_contacts`, `c_deals` — use for bulk queries or joining with financial data

**HubSpot object types:** `contacts`, `companies`, `deals`

---

## 4. Policies, Procedures & Reference Docs → **Confluence**

**Confluence site: https://fablefood.atlassian.net/wiki**
**Cloud ID: `9aaea3b1-0b23-49a9-8f86-a7f2f02ca748`**

| Space | Key | What lives here |
|---|---|---|
| **Finance** | `FIN` | Company registration, bank accounts, payroll, ESOP, Xero/Fishbowl integration guides, end-of-month procedures, investment rounds (SAFE, Seed), management reporting |
| **Sales Resources** | `SR` | Sales methodology (CENTRAL RESOURCE HUB), buyer personas, ICPs, discovery questions, product comparisons, sales cycle, prospecting guides |
| **Strategy** | `STRAT` | Annual strategy docs (2021, 2023+), OKRs by quarter and individual, market analysis (China), priority projects |
| **Customers** | `CUS` | Individual customer pages: Woolworths, Coles, Bidfood, HelloFresh (AU/UK), Sysco US, GFS US, Gousto, Marley Spoon, PFD, Compass Group, etc. Credit documentation |
| **Product Information** | `PI` | SKU specs (FB001–FB006), COAs, HS codes/tariffs/import duties, supply chain map, NPC process, food standards |
| **Logistics** | `LOG` | Full supply chain & ops procedures: AU/UK/US shipment processes, Fishbowl how-tos (sales orders, POs, manufacturing orders, inventory), Cartoncloud (AU), Peter Green/Americold/Mighty Plant/TFI (US/UK) logistics guides, import regulations, product recall plan, barcoding & relabelling, S&OP rolling forecast, FSVP entries (US), customs clearance, ocean freight, duty drawback |
| **Manufacturing** | `MAN` | Manufacturer directory (Hoshay/Everbest, Ahimsa, Dewina, Mixtio, Fast Fuel Meals) with contacts, product prices & quotes by SKU (FB001–FB026+), supplier contacts, non-conformance log, retort vendor details, manufacturing agreements, FDA facility IDs, carbon neutral certificates |
| **Legal** | `LEG` | Contract register (paginated, Pages 1–13+), contract templates (AU/UK/US/China), trade marks (word marks + image marks), patents (PCT applications: retorted mushrooms, GRF fat replacer, dehydrated mushroom myco-texturisation), packaging & labelling requirements (AU/UK/US/EU), insurance policies (AU/UK/US), IP assignment & licensing, health marketing claims |
| **Food Quality, Safety & Sustainability** | `FQS` | Product specifications (FB001–FB099), supplier QA program, approved suppliers list, food safety management system (HACCP/VACCP/TACCP), allergen incident management, non-conformance reporting (via Airtable), product recall procedures, testing programme, traceability, Fable policies, sustainability (EcoVadis, SEDEX, Pathzero, UN Global Compact), California Prop 65, cadmium limits |
| **People** | `PEOP` | HR policies, people management |
| **PR** | `PR` | PR, media, and communications |
| **Marketing** | `MARKETING` | Active marketing knowledge base (created Dec 2024) — brand, campaigns, content |
| **R&D – Shiitake Growing** | `RndShiitake` | Shiitake growing R&D project (created Jul 2025) |
| ~~[ARCHIVED] Marketing~~ | `MAR` | Archived — do not use for current info |

**Search tip:** Use `searchConfluenceUsingCql` or `searchAtlassian` before browsing. Example CQL: `space = "FIN" AND title ~ "payroll"`.

---

## 5. Team Communication & Real-Time Updates → **Slack**

**Slack workspace: fungideas.slack.com**

| Channel | Purpose |
|---|---|
| `#general` | Company-wide announcements |
| `#sales` | General sales discussion |
| `#sales-team` | Sales team internal chat |
| `#sales-pipeline-updates` | Pipeline movement and deal updates |
| `#sales-web-submission` | Inbound web leads |
| `#au-sales` | AU urgent orders & sampling questions |
| `#uk-sales` | UK urgent orders & sampling questions |
| `#us-sales` | US urgent orders, sampling, supplier setup |
| `#sales-shiitake-infused` | Shiitake-infused burger market development |
| `#au-fable-team` | Australia team channel |
| `#us-fable-team` | US team channel |
| `#press-social-media` | Internal PR discussions |
| `#logistics-au-sc360` | AU logistics shared channel with SC360 |
| `#hornbill` | Bookkeeper ↔ Fable communications |
| `#uk-hollandandbarrett` | Holland & Barrett UK customer channel |
| `#shared-fable-pr` | External PR agency shared channel |
| `#us-foraging-tours` | Fable Forage & Feast Tours USA |

**Search tip:** Use `slack_search_public_and_private` for message content, or `slack_search_channels` to find a relevant channel first.

---

## 6. Meeting Notes & Transcripts → **Fireflies**

**For summaries, action items, and discussions from recorded meetings.**

- Use `fireflies_get_transcripts` to list recent meetings
- Use `fireflies_search` to search by keyword/topic across all transcripts
- Use `fireflies_get_summary` for a specific meeting's action items and key points
- Use `fireflies_get_transcript` for the full meeting transcript

**Admin user:** Ji Davis (jidavis@fablefood.co)

---

## 7. External Email → **Gmail**

**For customer correspondence, supplier emails, and external communications.**

- Use `gmail_search_messages` with sender/recipient/subject filters
- Use `gmail_read_thread` to follow a full email conversation
- Best for: customer order confirmations, supplier negotiations, inbound enquiries

---

## Quick Decision Guide

| Question type | Go to |
|---|---|
| "What were our sales / revenue / costs?" | BigQuery → `xero_processed_data` |
| "What's our current stock / inventory?" | BigQuery → `fishbowl_datasources` |
| "What's in our sales pipeline?" | HubSpot CRM tools or `hubspot_datasources` |
| "What's our process / policy for X?" | Confluence → find the relevant space above |
| "What do we know about customer X?" | Confluence → Customers (CUS) space |
| "What are our product specs / SKUs?" | Confluence → Product Information (PI) or FQS (product specs FB001–FB099) |
| "What's our sales strategy / ICP?" | Confluence → Sales Resources (SR) |
| "Who manufactures our products / what do they cost?" | Confluence → Manufacturing (MAN) |
| "How do we ship / fulfil orders?" | Confluence → Logistics (LOG) |
| "What contracts / IP / insurance do we have?" | Confluence → Legal (LEG) |
| "What are our food safety or sustainability certifications?" | Confluence → FQS |
| "What did we discuss in our meeting about X?" | Fireflies → search transcripts |
| "What's being discussed on Slack about X?" | Slack → search channels above |
| "What did we email customer X?" | Gmail → search messages |

---

*Last updated: March 2026. Maintained by Ji Davis (jidavis@fablefood.co). Update this file when new spaces, channels, or datasets are added.*
