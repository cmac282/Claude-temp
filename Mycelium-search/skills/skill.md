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

**BigQuery workflow:** When unsure which table to query, call `list_datasets` → `list_tables` on the relevant dataset → `get_table_schema` on the specific table before writing SQL. Primary analytical dataset is `mycelium_ai`.

---

## 1. Financial Data → **BigQuery (Xero)**

**Best first stop for any financial question (revenue, costs, P&L, balance sheet, budget).**

### Key tables

| What you need | Table | Key columns |
|---|---|---|
| Monthly P&L by entity | `xero_processed_data.v_pnl_from_reports` | `entity`, `period` (DATE, first of month), `group`, `line`, `amountInCurrency`, `amountAUD`, `amountUSD`, `amountGBP`, `currency` |
| Balance sheet | `xero_processed_data.v_balance_sheet_from_reports` | `entity`, `period`, `category` (Assets/Liabilities/Equity), `group` (Bank/Accounts Receivable/Accounts Payable/Retained Earnings), `line`, `amountAUD`, `amountUSD`, `amountGBP` |
| Budget vs actuals | `xero_processed_data.v_budget_variance` | joins budget to actuals |
| Budget summary | `xero_processed_data.v_budget_summary_from_reports` | `entity`, `period`, `group` (Trading Income/Cost of Sales/Gross Profit/Operating Expenses/Net Profit), `line`, `amountAUD`, `amountUSD`, `amountGBP` |
| Raw invoices (AR & AP) | `xero_datasources.invoices_au` / `invoices_uk` / `invoices_us` | `invoiceid`, `invoicenumber`, `date`, `duedate`, `fullypaidondate`, `status`, `type` (ACCRECINV=receivable / ACCPAYINV=payable), `contact_name`, `contact_email`, `accountnumber` (AU-/UK-/US- prefix), `lineitems_description`, `lineitems_lineamount`, `lineitems_accountcode`, `lineitems_quantity`, `lineitems_unitamount`, `entity` |
| Credit notes | `xero_datasources.credit_notes_au` / `credit_notes_uk` / `credit_notes_us` | `id`, `creditnotenumber`, `date`, `status`, `contact_name`, `lineitems_lineamount`, `entity` |
| Historical P&L by year | `xero_datasources.pnl_YYYY_au` / `pnl_YYYY_uk` / `pnl_YYYY_us` | available from 2019 |

### Schema notes
- `entity` = `AU`, `UK`, or `US`
- P&L `group` values: `Income` (AU/UK) or `Revenue` (US) for sales; `Cost of Sales`; `Operating Expenses`; `Gross Profit`; `Net Profit`
- Sales lines: `line = 'Sales'` and `line = 'Sales - eCommerce'`
- AU entity: `amountUSD` is often 0 — use `amountAUD` and convert if needed
- Row amounts also in `amountInCurrency` (native currency of entity)

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

## 2. Inventory & Supply Chain → **BigQuery (`mycelium_ai` dataset)**

All cleaned Fishbowl tables live in the `mycelium_ai` dataset. Run `list_tables('mycelium_ai')` to see the full list.

### `mycelium_ai.t_inventory` — Current inventory snapshot (yesterday only)
One row per inventory tag/lot per location. Key columns:
`entity`, `partNum`, `partName`, `partActiveFlag`, `productNum`, `productDescription`, `productPrice`, `locationGroupName`, `location`, `qtyOnHand`, `qtyCommitted`, `batchNum`, `expiryDate`, `bestBeforeDate`, `shelfLife_expDate`, `shelfLife_BestBeforeDate`, `costAUD`, `costGBP`, `costUSD`, `partCostInCurrency`, `currency`, `pack_size`, `assetaccount`, `tagId`, `partId`, `productId`

- Available qty = `qtyOnHand - qtyCommitted`
- `shelfLife_expDate` buckets: `(1) < 0` expired · `(2) 0-99` · `(3) 100-199` · `(4) 200-299` · `(5) 300-399` · `(6) 400-499` · `(7) 500-599` · `(8) >= 600`

### `mycelium_ai.t_inventory_history` — Historical daily inventory (from 2022-12-05)
Same structure as t_inventory but with a `date` column. Use for trends and period comparisons.

### `mycelium_ai.t_sales` — Sales order line items (all live orders, yesterday snapshot)
One row per sales order line item. Key columns:
`entity`, `soNum`, `soLineItem`, `soDateCreated`, `soDateIssued`, `soDateFulfilled`, `soFulfillmentDate`, `dateShipped`, `monthShipped`, `weekShipped`, `customerName`, `customerNumber`, `locationGroupName`, `soShipToName`, `soShipToCity`, `partNum`, `productNum`, `description`, `customerPartNum`, `qtyFulfilled`, `qtyToFulfill`, `unitsShipped`, `unitsToShip`, `soItemUnitPrice`, `soItemTotalPrice`, `totalWeight`, `soCustomerPo`, `soVendorPO`, `soSalesman`, `partActiveFlag`, `customerActiveFlag`

### `mycelium_ai.t_purchase_orders` — Purchase order headers
`entity`, `ponumber`, `dateCreated`, `dateIssued`, `dateCompleted`, `postatus`, `vendorcontact`, `currency`, `totalTax`, `totalIncludesTax`, `carrier`, `username`, `paymentterms`, `shippingcountry`

### `mycelium_ai.t_purchase_order_items` — Purchase order line items
`entity`, `ponumber`, `poLineItem`, `description`, `poitemtype`, `dateScheduledFulfillment`, `qtyFulfilled`, `qtypicked`, `qtytofulfill`, `uom`, `qtyfulfilledkg`, `unitcost`, `totalcost`, `currency`
Join to `t_purchase_orders` on `ponumber`.

### `mycelium_ai.t_transfer_orders` — Transfer order headers
`entity`, `xonumber`, `dateCreated`, `dateIssued`, `dateCompleted`, `xostatus`, `xotype`, `countryfrom`, `countryto`, `carrier`, `fromAttn`, `shiptoAttn`

### `mycelium_ai.t_transfer_order_items` — Transfer order line items
`entity`, `xonumber`, `LineItem`, `description`, `xoitemtype`, `dateScheduledFulfillment`, `qtyFulfilled`, `qtypicked`, `qtytofulfill`, `uom`, `qtyfulfilledkg`
Join to `t_transfer_orders` on `xonumber`.

### `mycelium_ai.t_unit_price_history` — Monthly part cost history (month-end snapshots)
`entity`, `last_day_of_month`, `date`, `partNum`, `partName`, `avgCostInCurrency`, `currency`, `avgCostAUD`

### Raw Fishbowl datasources
Tables in `fishbowl_datasources` follow the pattern `tablename` (AU), `tablename_uk`, `tablename_us` — e.g. `fishbowl_datasources.so`, `so_uk`, `so_us`. Use `mycelium_ai` cleaned tables in preference.

---

## 3. CRM / Sales Pipeline → **BigQuery (`hubspot_datasources`) or HubSpot live tools**

### Live lookups
Use HubSpot MCP tools (`search_crm_objects`, `get_crm_objects`) for real-time deal, contact, and company lookups. Object types: `contacts`, `companies`, `deals`.

### Analytical queries: BigQuery `hubspot_datasources`

**`hubspot_datasources.c_deals`**
`hs_object_id`, `dealname`, `dealstage_name`, `pipeline_name`, `amount`, `amount_in_home_currency`, `deal_currency_code`, `closedate`, `end_deal_date`, `createdAt`, `hs_is_closed`, `hs_is_closed_won`, `hs_is_closed_lost`, `closed_won_reason`, `closed_lost_reason`, `hs_deal_stage_probability`, `hs_forecast_amount`, `hs_projected_amount`, `hs_acv`, `hs_arr`, `hs_mrr`, `hs_tcv`, `hs_days_to_close_raw`, `hs_is_stalled`, `hubspot_owner_name`, `hubspot_owner_assigneddate`, `hs_primary_associated_company`, `associations_contacts`, `associations_companies_count`, `associations_contacts_count`, `associations_emails_count`

**`hubspot_datasources.c_contacts`**
`hs_object_id`, `firstname`, `lastname`, `email`, `phone`, `mobilephone`, `jobtitle`, `industry`, `lifecyclestage`, `lead_source`, `hs_buying_role`, `company`, `company_website`, `website`, `associatedcompanyid`, `address`, `city`, `state`, `country`, `linkedin`, `linkedin_profile`, `createdAt`, `lastmodifieddate`

**`hubspot_datasources.c_companies`**
`hs_object_id`, `name`, `domain`, `website`, `industry`, `description`, `numberofemployees`, `hs_employee_range`, `no_outlets`, `annualrevenue`, `total_revenue`, `is_public`, `founded_year`, `address`, `city`, `state`, `country`, `timezone`, `phone`, `createdAt`, `updatedAt`

**`hubspot_datasources.t_calls`**
`hs_object_id`, `hs_call_title`, `hs_call_summary`, `hs_body_preview`, `hs_call_direction`, `hs_call_status`, `hs_call_duration`, `hs_call_has_transcript`, `hs_call_has_voicemail`, `hs_call_primary_deal`, `hs_call_deal_stage_during_call`, `hubspot_owner_name`, `associations_deals`, `createdAt`

**`hubspot_datasources.t_emails`**
`hs_object_id`, `hs_email_subject`, `hs_email_text`, `hs_body_preview`, `hs_email_sender_email`, `hs_email_direction`, `hs_email_status`, `hs_email_open_count`, `hs_email_reply_count`, `hs_email_click_count`, `hs_email_associated_contact_id`, `hubspot_owner_name`, `associations_deals`, `createdAt`

**`hubspot_datasources.t_meetings`**
`hs_object_id`, `hs_meeting_title`, `hs_meeting_body`, `hs_meeting_start_time`, `hs_internal_meeting_notes`, `hs_body_preview`, `hubspot_owner_name`, `associations_deals`, `createdAt`

**`hubspot_datasources.t_notes`**
`hs_object_id`, `hs_note_body`, `hs_body_preview`, `hubspot_owner_name`, `associations_deals`, `createdAt`

---

## 4. Policies, Procedures & Reference Docs → **Confluence**

**Site: https://fablefood.atlassian.net/wiki** · **Cloud ID: `9aaea3b1-0b23-49a9-8f86-a7f2f02ca748`**

### How to search and cite
1. **Search first**: use `search_pages` with CQL before browsing
2. **Get page content**: use `get_page` with the page ID
3. **Check attachments**: use `get_page_attachments` then `get_attachment_content` — contracts, price lists, specs and SOPs are often attachments not inline text
4. **Cite with URL**: `https://fablefood.atlassian.net/wiki/spaces/{KEY}/pages/{PAGE_ID}`
5. **Explore hierarchy**: use `get_page_children` for spaces with nested structures

### CQL examples
```
space = "FIN" AND title ~ "payroll"
space = "CUS" AND title ~ "HelloFresh"
text ~ "FB003" AND space = "FQS"
text ~ "Sysco" AND space = "LEG"
space = "MAN" AND title ~ "Hoshay"
```

### Spaces

| Space | Key | What lives here | Attachment note |
|---|---|---|---|
| **Finance** | `FIN` | Company registration, bank accounts, payroll, ESOP, Xero/Fishbowl integration guides, end-of-month procedures, investment rounds (SAFE, Seed), management reporting | Financial models and reports often as Excel/PDF |
| **Sales Resources** | `SR` | Sales methodology (CENTRAL RESOURCE HUB), buyer personas, ICPs, discovery questions, product comparisons, sales cycle, prospecting guides | Slide decks and battle cards often as PPTX/PDF |
| **Strategy** | `STRAT` | Annual strategy docs (2021, 2023+), OKRs by quarter and individual, market analysis (China), priority projects | — |
| **Customers** | `CUS` | Individual customer pages: Woolworths, Coles, Bidfood, HelloFresh (AU/UK), Sysco US, GFS US, Gousto, Marley Spoon, PFD, Compass Group, etc. Credit documentation | Contract copies and price agreements attached per customer |
| **Product Information** | `PI` | SKU specs (FB001–FB006), COAs, HS codes/tariffs/import duties, supply chain map, NPC process, food standards | COA and spec sheets as PDF |
| **Logistics** | `LOG` | AU/UK/US shipment processes, Fishbowl how-tos, Cartoncloud (AU), Peter Green/Americold/Mighty Plant/TFI (US/UK) logistics guides, import regulations, product recall plan, barcoding & relabelling, S&OP rolling forecast, FSVP entries (US), customs clearance, ocean freight, duty drawback | SOPs and process guides as Word/PDF |
| **Manufacturing** | `MAN` | Manufacturer directory (Hoshay/Everbest, Ahimsa, Dewina, Mixtio, Fast Fuel Meals) with contacts, product prices & quotes by SKU (FB001–FB026+), non-conformance log, retort vendors, manufacturing agreements, FDA facility IDs, carbon neutral certificates | **Price sheets and quotes attached per manufacturer — always check attachments for current pricing** |
| **Legal** | `LEG` | Contract register (Pages 1–13+), contract templates (AU/UK/US/China), trade marks, patents (PCT: retorted mushrooms, GRF fat replacer, dehydrated mushroom myco-texturisation), packaging & labelling requirements, insurance policies, IP assignment & licensing, health marketing claims | **Executed contracts as signed PDFs — use `get_page_attachments` + `get_attachment_content`** |
| **Food Quality, Safety & Sustainability** | `FQS` | Product specifications (FB001–FB099), supplier QA program, approved suppliers list, HACCP/VACCP/TACCP, allergen management, non-conformance reporting, product recall procedures, traceability, sustainability (EcoVadis, SEDEX, Pathzero, UN Global Compact), California Prop 65, cadmium limits | **Full product specs as PDF — search by SKU code** |
| **People** | `PEOP` | HR policies, people management | — |
| **PR** | `PR` | PR, media, and communications | Press releases often attached |
| **Marketing** | `MARKETING` | Active marketing knowledge base (created Dec 2024) — brand, campaigns, content | — |
| **R&D – Shiitake Growing** | `RndShiitake` | Shiitake growing R&D project (created Jul 2025) | — |
| ~~[ARCHIVED] Marketing~~ | `MAR` | Archived — do not use | — |

---

## 5. Team Communication & Real-Time Updates → **Slack**

**Workspace: fungideas.slack.com**

Use `search_messages` with keywords. Use `read_thread` for full reply context. Cite channel name, sender, and date.

### Sales & Pipeline
| Channel | Purpose |
|---|---|
| `#sales` | General sales discussion |
| `#sales-team` | Sales team internal chat |
| `#sales-pipeline-updates` | Pipeline movement and deal updates |
| `#au-sales` | AU urgent orders & sampling |
| `#uk-sales` | UK urgent orders & sampling |
| `#us-sales` | US urgent orders, sampling, supplier setup |
| `#prospecting` | Prospecting discussion |
| `#sales-shiitake-infused` | Shiitake-infused burger market development |

### Customer-specific channels
| Channel | Customer / Purpose |
|---|---|
| `#au-woolworths` | Woolworths AU |
| `#au-marley-spoon` | Marley Spoon AU |
| `#au-compass-group` | Compass Group AU |
| `#au-sg-gyg` | Guzman y Gomez (SG/AU) |
| `#au-shiitake-infused` | AU shiitake-infused product |
| `#au-zambrero` | Zambrero AU |
| `#au-zeus-street-greek` | Zeus Street Greek AU |
| `#au-foraging-tours` | Fable Forage & Feast Tours AU |
| `#uk-gousto` | Gousto UK (frozen braised mushroom bags) |
| `#uk-wagamama` | Wagamama UK |
| `#uk-hollandandbarrett` | Holland & Barrett UK (retail packs) |
| `#uk-finnebrogue` | Finnebrogue UK (technical due diligence) |
| `#uk-europe-hello-fresh` | HelloFresh / Green Chef UK/Europe |
| `#uk-bidfood` | Bidfood UK |
| `#uk-cote` | Côte Brasserie UK |
| `#uk-bigtablegroup` | Big Table Group UK (Bella Italia, Las Iguanas, etc.) |
| `#uk-tradeshows` | UK trade show coordination |
| `#us-foraging-tours` | Fable Forage & Feast Tours USA |
| `#us-hellofresh` | HelloFresh US |
| `#us-compass-group` | Compass Group US |
| `#us-just-salad` | Just Salad US |
| `#us-purple-carrot` | Purple Carrot US |
| `#us-planta` | Planta US |
| `#us-central-market-heb` | Central Market / HEB US |
| `#us-platinum-quesada` | Platinum/Quesada US |
| `#us-shiitake-infused` | US shiitake-infused product |
| `#customer-feedback` | Customer comments from emails, messages, social |
| `#customer-reports` | Customer complaints, mould tracking, NCR |

### Operations & Logistics
| Channel | Purpose |
|---|---|
| `#logistics_internal` | Internal logistics discussion |
| `#logistics-au-sc360` | AU logistics shared channel with SC360 |
| `#logistics-usa-threadline-ops` | US logistics with Your Logistics (YL) and SC360 |
| `#uk-supply-chain-and-orders` | UK supply chain and order management |
| `#fishbowl_inventory_management_system` | Fishbowl IMS discussion |
| `#forecasts` | Demand and sales forecasts |
| `#riverlogic` | Riverlogic supply chain planning |

### Finance
| Channel | Purpose |
|---|---|
| `#finance` | Finance team discussion |
| `#finops-team` | FinOps daily standup and discussion |
| `#hornbill` | Bookkeeper ↔ Fable communications |
| `#grants-and-free-cash` | Grants and accelerators |

### Product, Quality & R&D
| Channel | Purpose |
|---|---|
| `#product-and-packaging-specifications` | Product and packaging specs |
| `#sampling_npd` | Sampling and new product development |
| `#food-safety` | Food safety |
| `#quality-assurance` | QA |
| `#rnd-dropbox-public-suggestions` | R&D suggestions |
| `#lean-six-sigma-training` | Lean Six Sigma training |

### Marketing & PR
| Channel | Purpose |
|---|---|
| `#marketing` | Marketing team |
| `#marketing-comms-team` | Marketing comms |
| `#press-social-media` | Internal PR discussion |
| `#industry-news-links` | Industry news and Instagram posts |
| `#competitive-research` | Competitor activity |
| `#customer-research` | Customer research |

### Legal
| Channel | Purpose |
|---|---|
| `#legal` | Legal team discussion |

### Team & Culture
| Channel | Purpose |
|---|---|
| `#general` | Company-wide announcements |
| `#random` | Non-work banter |
| `#au-fable-team` | Australia team |
| `#health-team` | Health team |
| `#culture-club` | Culture club |
| `#leave` | Leave requests and approvals |
| `#team-gtm-uk` | UK go-to-market team |
| `#share-the-shroom` | Mushroom inspiration, cook-ups, geeky content |
| `#puzzle-connections` | NYT Connections puzzle |

---

## 6. Meeting Notes & Transcripts → **Fireflies**

| Task | Tool |
|---|---|
| List recent meetings | `fireflies_get_transcripts` |
| Search by topic/keyword across all meetings | `fireflies_search` |
| Get action items and key points | `fireflies_get_summary` |
| Read full transcript | `fireflies_get_transcript` |

**Admin user:** Ji Davis (jidavis@fablefood.co). Always cite: meeting title, date, and Fireflies transcript ID.

---

## 7. External Email → **Gmail**

- `search_emails`: filter by `from:`, `to:`, `subject:`, date range
- `get_email`: read a full email thread
- Defaults to **external-only** (filters out @fablefood.co)
- Use `user_email` parameter to read any team member's inbox

Best for: customer order confirmations, supplier negotiations, inbound enquiries, distributor/retailer threads.

---

## Quick Decision Guide

| Question | Go to |
|---|---|
| Sales / revenue / costs / P&L | BigQuery → `xero_processed_data.v_pnl_from_reports` |
| Balance sheet / assets / liabilities | BigQuery → `xero_processed_data.v_balance_sheet_from_reports` |
| Budget vs actuals | BigQuery → `xero_processed_data.v_budget_variance` |
| Invoices / accounts payable / receivable | BigQuery → `xero_datasources.invoices_au/uk/us` |
| Credit notes | BigQuery → `xero_datasources.credit_notes_au/uk/us` |
| Current stock / inventory levels | BigQuery → `mycelium_ai.t_inventory` |
| Historical inventory trends | BigQuery → `mycelium_ai.t_inventory_history` |
| Open / fulfilled sales orders | BigQuery → `mycelium_ai.t_sales` |
| Purchase orders / supplier orders | BigQuery → `mycelium_ai.t_purchase_orders` + `t_purchase_order_items` |
| Stock transfers between locations | BigQuery → `mycelium_ai.t_transfer_orders` + `t_transfer_order_items` |
| Part cost history over time | BigQuery → `mycelium_ai.t_unit_price_history` |
| Sales pipeline / deals | HubSpot tools or `hubspot_datasources.c_deals` |
| CRM contacts or companies | HubSpot tools or `hubspot_datasources.c_contacts` / `c_companies` |
| CRM call / email / meeting logs | `hubspot_datasources.t_calls` / `t_emails` / `t_meetings` |
| Process / policy for X | Confluence → find relevant space above |
| Customer account info | Confluence → Customers (`CUS`) + check attachments |
| Product specs / SKUs | Confluence → Product Info (`PI`) or FQS (FB001–FB099) + attachments |
| Sales strategy / ICP | Confluence → Sales Resources (`SR`) |
| Manufacturer pricing (FB00X) | Confluence → Manufacturing (`MAN`) — **check page attachments** |
| Shipping / fulfilment procedures | Confluence → Logistics (`LOG`) |
| Contracts / IP / insurance | Confluence → Legal (`LEG`) — **check page attachments** |
| Food safety / sustainability certs | Confluence → FQS |
| Meeting discussions about X | Fireflies → `fireflies_search` |
| Slack discussions about X | Slack → `search_messages` + channels above |
| External emails with customer/supplier X | Gmail → `search_emails` |
| BigQuery table schema | BigQuery → `get_table_schema` |

---

## Citing sources

- **BigQuery**: full table name (e.g. `` `mycelium_ai.t_inventory` ``) + query used
- **Confluence page**: title + URL (`https://fablefood.atlassian.net/wiki/spaces/{KEY}/pages/{ID}`)
- **Confluence attachment**: filename + parent page URL
- **Slack**: channel name, sender, approximate date
- **Fireflies**: meeting title, date, transcript ID
- **Gmail**: sender, recipient, subject, date

---

*Last updated: March 2026. Maintained by Ji Davis (jidavis@fablefood.co). Update when new spaces, channels, datasets, or connectors are added.*
