---
name: mycelium-search
description: "Knowledge map for Fable Food: tells Claude where to find information across all connected tools and systems. Load this at the start of any research or data task."
---

# Fable Food — Information Context Schema

This document tells Claude exactly where to look when answering questions for the Fable Food team. Before starting any multi-step research or data task, refer to the relevant section below to identify the right tool and location. **Always cite your sources** (table names, Confluence page URLs, Slack channels, meeting IDs) when answering.

---

## Connector Overview

| Connector | Tools available |
|---|---|
| **BigQuery** | `execute_sql`, `list_datasets`, `list_tables`, `get_table_schema` |
| **Confluence** | `search_pages`, `get_page`, `list_spaces`, `get_space_pages`, `get_page_children`, `get_page_comments`, `get_page_attachments`, `get_attachment_content` |
| **Slack** | `search_messages`, `search_files`, `list_channels`, `read_channel`, `read_thread`, `get_user`, `search_users` |
| **Gmail** | `search_emails`, `list_emails`, `get_email` |
| **Fireflies** | `fireflies_get_transcripts`, `fireflies_search`, `fireflies_get_summary`, `fireflies_get_transcript` |

**BigQuery workflow:** When you are unsure which table to query, first call `list_datasets` to see all available datasets, then `list_tables` on the relevant dataset, then `get_table_schema` on the specific table before writing SQL. The primary analytical dataset is `mycelium_ai`.

---

## 1. Financial Data → **BigQuery (Xero)**

**Best first stop for any financial question (revenue, costs, P&L, balance sheet, budget).**

### Key tables

| What you need | Table | Key columns |
|---|---|---|
| Monthly P&L by entity | `xero_processed_data.v_pnl_from_reports` | `entity`, `period` (DATE, first of month), `group`, `line`, `amountInCurrency`, `amountAUD`, `amountUSD`, `amountGBP`, `currency` |
| Balance sheet | `xero_processed_data.v_balance_sheet_from_reports` | `entity`, `period`, `category`, `group`, `line`, `amountAUD/USD/GBP` |
| Budget vs actuals | `xero_processed_data.v_budget_variance` | joins budget to actuals |
| Budget summary | `xero_processed_data.v_budget_summary_from_reports` | mirrors P&L structure with budgeted amounts |
| Raw invoices | `xero_datasources.invoices_au`, `invoices_uk`, `invoices_us` | `invoiceid`, `invoicenumber`, `date`, `duedate`, `fullypaidondate`, `status`, `type` (ACCRECINV=receivable / ACCPAYINV=payable), `contact_name`, `contact_email`, `accountnumber`, `lineitems_description`, `lineitems_lineamount`, `lineitems_accountcode`, `entity` |
| Credit notes | `xero_datasources.credit_notes_au/uk/us` | `id`, `creditnotenumber`, `date`, `status`, `contact_name`, `lineitems_lineamount`, `entity` |
| Historical P&L by year | `xero_datasources.pnl_YYYY_au/uk/us` | available from 2019 |

### Schema notes
- `entity` = `AU`, `UK`, or `US`
- `group` values: `Income` (AU/UK) or `Revenue` (US) for top-line sales; `Cost of Sales`; `Operating Expenses`; `Gross Profit`; `Net Profit`
- Sales lines: `line = 'Sales'` and `line = 'Sales - eCommerce'`
- AU entity: `amountUSD` is often 0 — use `amountAUD` and convert if needed
- Amounts also available as `amountInCurrency` (native currency)

### Example query
```sql
SELECT entity, period, SUM(amountInCurrency) AS sales, currency
FROM `xero_processed_data.v_pnl_from_reports`
WHERE `group` IN ('Income', 'Revenue') AND line = 'Sales'
  AND period >= DATE_TRUNC(DATE_SUB(CURRENT_DATE(), INTERVAL 6 MONTH), MONTH)
GROUP BY entity, period, currency
ORDER BY period, entity
```

---

## 2. Inventory & Supply Chain → **BigQuery (Fishbowl — `mycelium_ai` dataset)**

**For stock levels, sales orders, product/part data, purchase orders, and transfers.**

All cleaned Fishbowl tables live in the `mycelium_ai` dataset. Run `list_tables('mycelium_ai')` to see the full list.

| What you need | Table | Key columns / notes |
|---|---|---|
| Current inventory (yesterday snapshot) | `mycelium_ai.t_inventory` | `entity`, `partNum`, `partName`, `locationGroupName`, `qtyOnHand`, `qtyCommitted`, `batchNum`, `expiryDate`, `bestBeforeDate`, `shelfLife_expDate`, `costAUD`, `productPrice`, `pack_size`. **One row per tag/lot per location.** |
| Historical daily inventory (from 2022-12-05) | `mycelium_ai.t_inventory_history` | `date`, `entity`, `partNum`, `partName`, `locationGroupName`, `qtyOnHand`, `qtyCommitted`, `costAUD`. Use for trends. |
| Sales order line items | `mycelium_ai.t_sales` | `entity`, `soNumber`, `customer`, `orderDate`, `shipDate`, `status`, `partNum`, `partName`, `qtyOrdered`, `qtyFulfilled`, `unitPrice`, `totalPrice`, `currency`. Snapshot = all live orders as of yesterday. |
| Purchase order headers | `mycelium_ai.t_purchase_orders` | `entity`, `ponumber`, `dateCreated`, `dateIssued`, `dateCompleted`, `postatus`, `vendor`, `currency`, `totalcost`. |
| Purchase order line items | `mycelium_ai.t_purchase_order_items` | `entity`, `ponumber`, `description`, `qtyFulfilled`, `qtytofulfill`, `uom`, `qtyfulfilledkg`, `unitcost`, `totalcost`, `currency`. Join to `t_purchase_orders` on `ponumber`. |
| Transfer order headers | `mycelium_ai.t_transfer_orders` | `entity`, `xonumber`, `dateCreated`, `dateIssued`, `dateCompleted`, `xostatus`, `xotype`, `countryfrom`, `countryto`, `carrier`. |
| Transfer order line items | `mycelium_ai.t_transfer_order_items` | `entity`, `xonumber`, `description`, `qtyFulfilled`, `qtytofulfill`, `uom`, `qtyfulfilledkg`. Join to `t_transfer_orders` on `xonumber`. |
| Monthly part cost history | `mycelium_ai.t_unit_price_history` | `entity`, `last_day_of_month`, `partNum`, `partName`, `avgCostInCurrency`, `currency`, `avgCostAUD`. Month-end snapshots only. |

### Inventory notes
- `shelfLife_expDate` buckets: `(1) < 0` = expired; `(2) 0-99` days; up to `(8) >= 600` days
- Available qty = `qtyOnHand - qtyCommitted`
- Tables in raw Fishbowl datasources (`fishbowl_datasources`) use the pattern `tablename`, `tablename_uk`, `tablename_us` per region (e.g. `fishbowl_datasources.so`, `so_uk`, `so_us`)
- US entity pack sizes are converted from lb to kg (× 0.45359237)

---

## 3. CRM / Sales Pipeline → **BigQuery (HubSpot) or HubSpot live tools**

**For contacts, companies, deals, pipeline stage, and engagement history.**

### Live lookups: HubSpot MCP tools
Use `search_crm_objects` / `get_crm_objects` for real-time deal, contact, and company lookups.

### Bulk / analytical queries: BigQuery `hubspot_datasources`

| What you need | Table | Key columns |
|---|---|---|
| Deals / pipeline | `hubspot_datasources.c_deals` | `hs_object_id`, `dealname`, `amount`, `dealstage`, `pipeline`, `closedate`, `createdate`, `hubspot_owner_name`, `associations_companies`, `associations_contacts` |
| Contacts | `hubspot_datasources.c_contacts` | `hs_object_id`, `firstname`, `lastname`, `email`, `company`, `jobtitle`, `lifecyclestage`, `hubspot_owner_name`, `associations_companies` |
| Companies | `hubspot_datasources.c_companies` | `hs_object_id`, `name`, `domain`, `industry`, `country`, `annualrevenue`, `hs_num_associated_deals`, `hubspot_owner_name` |
| Calls | `hubspot_datasources.t_calls` | `hs_object_id`, `hs_call_title`, `hs_call_body`, `hs_call_duration`, `hs_call_direction`, `hubspot_owner_name`, `associations_deals`, `createdAt` |
| Emails | `hubspot_datasources.t_emails` | `hs_object_id`, `hs_email_subject`, `hs_email_text`, `hs_email_direction`, `hubspot_owner_name`, `associations_deals`, `createdAt` |
| Meetings | `hubspot_datasources.t_meetings` | `hs_object_id`, `hs_meeting_title`, `hs_meeting_body`, `hs_meeting_start_time`, `hubspot_owner_name`, `associations_deals` |
| Notes | `hubspot_datasources.t_notes` | `hs_object_id`, `hs_note_body`, `hubspot_owner_name`, `associations_deals`, `createdAt` |

**HubSpot object types for live tools:** `contacts`, `companies`, `deals`

---

## 4. Policies, Procedures & Reference Docs → **Confluence**

**Confluence site: https://fablefood.atlassian.net/wiki**
**Cloud ID: `9aaea3b1-0b23-49a9-8f86-a7f2f02ca748`**

### How to search and cite Confluence
1. **Always search first**: use `search_pages` with CQL (e.g. `space = "FIN" AND title ~ "payroll"`) before browsing
2. **Get the page**: use `get_page` with the page ID to retrieve full content
3. **Check for attachments**: use `get_page_attachments` — many key documents (price lists, contracts, specs) are attachments, not inline text. Use `get_attachment_content` to read them.
4. **Always return the page URL** when citing Confluence content (format: `https://fablefood.atlassian.net/wiki/spaces/{KEY}/pages/{PAGE_ID}`)
5. **Check child pages**: use `get_page_children` for spaces with nested structures (e.g. Customers, Manufacturing, Legal)

### Confluence spaces

| Space | Key | What lives here | Attachment note |
|---|---|---|---|
| **Finance** | `FIN` | Company registration, bank accounts, payroll, ESOP, Xero/Fishbowl integration guides, end-of-month procedures, investment rounds (SAFE, Seed), management reporting | Financial models and reports often attached as Excel/PDF |
| **Sales Resources** | `SR` | Sales methodology (CENTRAL RESOURCE HUB), buyer personas, ICPs, discovery questions, product comparisons, sales cycle, prospecting guides | Slide decks and battle cards often attached as PPTX/PDF |
| **Strategy** | `STRAT` | Annual strategy docs (2021, 2023+), OKRs by quarter and individual, market analysis (China), priority projects | — |
| **Customers** | `CUS` | Individual customer pages: Woolworths, Coles, Bidfood, HelloFresh (AU/UK), Sysco US, GFS US, Gousto, Marley Spoon, PFD, Compass Group, etc. Credit documentation | Contract copies and price agreements often attached per customer |
| **Product Information** | `PI` | SKU specs (FB001–FB006), COAs, HS codes/tariffs/import duties, supply chain map, NPC process, food standards | COA and specification sheets attached as PDF |
| **Logistics** | `LOG` | Full supply chain & ops procedures: AU/UK/US shipment processes, Fishbowl how-tos (sales orders, POs, manufacturing orders, inventory), Cartoncloud (AU), Peter Green/Americold/Mighty Plant/TFI (US/UK) logistics guides, import regulations, product recall plan, barcoding & relabelling, S&OP rolling forecast, FSVP entries (US), customs clearance, ocean freight, duty drawback | SOPs and process guides often attached as Word/PDF |
| **Manufacturing** | `MAN` | Manufacturer directory (Hoshay/Everbest, Ahimsa, Dewina, Mixtio, Fast Fuel Meals) with contacts, product prices & quotes by SKU (FB001–FB026+), supplier contacts, non-conformance log, retort vendor details, manufacturing agreements, FDA facility IDs, carbon neutral certificates | Price sheets and quotes attached per manufacturer — always check attachments for current pricing |
| **Legal** | `LEG` | Contract register (paginated, Pages 1–13+), contract templates (AU/UK/US/China), trade marks, patents (PCT applications: retorted mushrooms, GRF fat replacer, dehydrated mushroom myco-texturisation), packaging & labelling requirements (AU/UK/US/EU), insurance policies, IP assignment & licensing, health marketing claims | Executed contracts attached as signed PDFs — use `get_page_attachments` + `get_attachment_content` to read them |
| **Food Quality, Safety & Sustainability** | `FQS` | Product specifications (FB001–FB099), supplier QA program, approved suppliers list, food safety management system (HACCP/VACCP/TACCP), allergen incident management, non-conformance reporting (via Airtable), product recall procedures, testing programme, traceability, sustainability (EcoVadis, SEDEX, Pathzero, UN Global Compact), California Prop 65, cadmium limits | Full product specs attached as PDF — search by SKU code |
| **People** | `PEOP` | HR policies, people management | — |
| **PR** | `PR` | PR, media, and communications | Press releases often attached |
| **Marketing** | `MARKETING` | Active marketing knowledge base (created Dec 2024) — brand, campaigns, content | — |
| **R&D – Shiitake Growing** | `RndShiitake` | Shiitake growing R&D project (created Jul 2025) | — |
| ~~[ARCHIVED] Marketing~~ | `MAR` | Archived — do not use for current info | — |

### CQL examples
```
# Find payroll procedures in Finance space
space = "FIN" AND title ~ "payroll"

# Find HelloFresh customer page
space = "CUS" AND title ~ "HelloFresh"

# Find FB003 product spec
text ~ "FB003" AND space = "FQS"

# Find contracts containing "Sysco"
text ~ "Sysco" AND space = "LEG"
```

---

## 5. Team Communication & Real-Time Updates → **Slack**

**Workspace: fungideas.slack.com**

Use `search_messages` with keywords to find relevant discussions. Always include the channel name and message timestamp when citing Slack. Use `read_thread` to get the full context of a reply thread.

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

**Tip:** Use `list_channels` if you need to discover channels not listed here.

---

## 6. Meeting Notes & Transcripts → **Fireflies**

**For summaries, action items, and discussions from recorded meetings.**

| Task | Tool |
|---|---|
| List recent meetings | `fireflies_get_transcripts` |
| Search by topic/keyword across all meetings | `fireflies_search` |
| Get action items and key points for a specific meeting | `fireflies_get_summary` |
| Read full transcript | `fireflies_get_transcript` |

**Admin user:** Ji Davis (jidavis@fablefood.co)

Always cite: meeting title, date, and Fireflies transcript ID when referencing meeting content.

---

## 7. External Email → **Gmail**

**For customer correspondence, supplier emails, and external communications.**

- `search_emails`: filter by `from:`, `to:`, `subject:`, date range
- `list_emails`: browse recent emails for a mailbox
- `get_email`: read a full email thread
- Connector defaults to **external-only** emails (filters out internal @fablefood.co)
- Use `user_email` parameter to read any team member's inbox (domain-wide delegation enabled)

**Best for:** customer order confirmations, supplier negotiations, inbound enquiries, email threads with distributors/retailers.

---

## Quick Decision Guide

| Question type | Go to |
|---|---|
| "What were our sales / revenue / costs?" | BigQuery → `xero_processed_data.v_pnl_from_reports` |
| "What's our current stock / inventory?" | BigQuery → `mycelium_ai.t_inventory` |
| "How has inventory changed over time?" | BigQuery → `mycelium_ai.t_inventory_history` |
| "What sales orders are open?" | BigQuery → `mycelium_ai.t_sales` |
| "What have we ordered from suppliers?" | BigQuery → `mycelium_ai.t_purchase_orders` + `t_purchase_order_items` |
| "What stock transfers are in progress?" | BigQuery → `mycelium_ai.t_transfer_orders` + `t_transfer_order_items` |
| "How have part costs changed?" | BigQuery → `mycelium_ai.t_unit_price_history` |
| "What's in our sales pipeline?" | HubSpot CRM tools or `hubspot_datasources.c_deals` |
| "What's our process / policy for X?" | Confluence → find the relevant space above |
| "What do we know about customer X?" | Confluence → Customers (`CUS`) space + check attachments |
| "What are our product specs / SKUs?" | Confluence → Product Information (`PI`) or FQS (specs FB001–FB099) + attachments |
| "What's our sales strategy / ICP?" | Confluence → Sales Resources (`SR`) |
| "What does manufacturer X charge for FB00Y?" | Confluence → Manufacturing (`MAN`) — check page attachments for price sheets |
| "How do we ship / fulfil orders?" | Confluence → Logistics (`LOG`) |
| "What contracts / IP / insurance do we have?" | Confluence → Legal (`LEG`) — check page attachments for executed contracts |
| "What are our food safety or sustainability certifications?" | Confluence → FQS |
| "What did we discuss in our meeting about X?" | Fireflies → `fireflies_search` |
| "What's being discussed on Slack about X?" | Slack → `search_messages` then check channels above |
| "What did we email customer X?" | Gmail → `search_emails` |
| "What's the schema for BigQuery table Y?" | BigQuery → `get_table_schema` |

---

## Citing sources

Always include source references in your answers:
- **BigQuery**: cite the full table name (e.g. `` `mycelium_ai.t_inventory` ``) and the query used
- **Confluence**: include the page title and URL (`https://fablefood.atlassian.net/wiki/spaces/{KEY}/pages/{ID}`)
- **Confluence attachments**: include attachment filename and the parent page URL
- **Slack**: include channel name, sender, and approximate date
- **Fireflies**: include meeting title, date, and transcript ID
- **Gmail**: include sender, recipient, subject, and date

---

*Last updated: March 2026. Maintained by Ji Davis (jidavis@fablefood.co). Update this file when new spaces, channels, datasets, or connectors are added.*
