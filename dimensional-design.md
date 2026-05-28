# Dimensional &amp; Data Warehouse Design Standards

> The Kimball-style modeling standards that turn a curated lake into a semantic
> layer business users can actually reason about. These are the rules I apply when
> designing the **Gold** layer and the certified semantic models on top of it.

---

## 1. Star schema is the default

Every reporting subject area is modeled as a **star**: one fact table surrounded by
conformed dimensions, joined on surrogate keys. Snowflaking is used sparingly and
only where a dimension is genuinely large and shared (e.g. a deep product
hierarchy); otherwise dimensions are kept flat for query performance and
business-user comprehension.

```mermaid
erDiagram
    dim_date        ||--o{ fact_invoice : invoice_date_key
    dim_customer    ||--o{ fact_invoice : customer_key
    dim_product     ||--o{ fact_invoice : product_key
    dim_geography   ||--o{ fact_invoice : geography_key
    dim_channel     ||--o{ fact_invoice : channel_key

    fact_invoice {
        int    invoice_line_key PK
        int    invoice_date_key FK
        int    customer_key FK
        int    product_key FK
        int    geography_key FK
        int    channel_key FK
        string invoice_no DD
        decimal gross_revenue_amt
        decimal discount_amt
        decimal net_revenue_amt
        decimal cost_amt
        int    quantity
    }
    dim_customer {
        int    customer_key PK
        string customer_id "natural key"
        string customer_name
        string segment "SCD2"
        date   valid_from
        date   valid_to
        bool   is_current
    }
    dim_date {
        int    date_key PK
        date   date
        int    year
        int    quarter
        int    month
        string month_name
        int    fiscal_year
        bool   is_weekday
    }
```

---

## 2. The four-step dimensional design method

For every new subject area:

1. **Select the business process** — what event are we measuring? (e.g. *invoicing*,
   *settlement*, *claim adjudication*).
2. **Declare the grain** — the most atomic level one fact row represents (e.g.
   *one invoice line*). Declared explicitly and never mixed.
3. **Identify the dimensions** — the "by what" of analysis (by date, customer,
   product, geography, channel...).
4. **Identify the facts** — the numeric measures at that grain (gross/net revenue,
   cost, quantity). Only facts true at the declared grain belong here.

---

## 3. Fact table standards

| Standard | Rule |
|----------|------|
| Grain | One declared grain per fact; documented in the table description |
| Keys | Foreign keys are **surrogate** keys to dimensions, never natural keys |
| Additivity | Mark each measure additive / semi-additive / non-additive |
| Degenerate dims | Transaction IDs (invoice no.) live on the fact as degenerate dimensions |
| Null measures | Use `0`/`NULL` deliberately; document which |
| Fact types | Transaction, periodic snapshot, or accumulating snapshot — chosen per process |

**Semi-additive measures** (e.g. balances, headcount) are flagged because they sum
across some dimensions but not across time — the semantic model handles them with
`LASTNONBLANK`-style measures rather than naive `SUM`.

---

## 4. Dimension table standards

| Standard | Rule |
|----------|------|
| Surrogate key | Integer/hash PK, source-independent |
| Natural key | Retained as an attribute for lineage/reconciliation |
| SCD policy | Declared per attribute (see [layers doc](../docs/data-platform-layers.md)) |
| Conformed | Shared dimensions built once, reused across facts |
| Role-playing | Modeled once, referenced multiple times (Date as order/ship/invoice) |
| Unknown member | A `-1` "Unknown" row so facts with missing keys still join |

The **Unknown member** pattern matters: early-arriving facts with a not-yet-loaded
dimension key point at `-1` rather than disappearing from totals.

---

## 5. Handling many-to-many

Many-to-many relationships (e.g. customer ↔ account, transaction ↔ product
bundle) are resolved with a **bridge table** at the correct grain, with a documented
allocation/weighting rule where the relationship would otherwise double-count.
Bidirectional cross-filtering is used surgically (and tested) rather than switched
on globally, because global bi-di filtering creates ambiguous paths and slow
queries.

---

## 6. From Gold star schema to semantic model

The Gold star schema maps almost 1:1 to the Power BI / Direct Lake semantic model:

- Relationships: single-direction, on surrogate keys, `Many → One` from fact to dim.
- Storage: **Direct Lake** for facts/large dims; Import for tiny static dims.
- A dedicated, contiguous **Date dimension** marked as the model's date table to
  enable time-intelligence DAX.
- Hidden technical columns (surrogate keys, SCD flags) so the field list shows only
  business-meaningful attributes.
- **Calculation groups** for time intelligence and formatting (see
  [Project 1](../../01-Microsoft-Fabric-Enterprise-Analytics/dax/calculation-groups.dax))
  so YTD/MTD/QoQ logic is written once, not per measure.

---

## 7. Documentation deliverables per subject area

Every modeled subject area ships with: a grain statement, an ER diagram, a
source-to-target mapping, the SCD policy per dimension, the measure catalog with
additivity flags, and the certified KPI definitions
([KPI framework](kpi-framework.md)). This is what makes a model **certifiable** and
audit-ready rather than just functional.
