# ZatiqGO — Complete Workflow

A simple guide to how ZatiqGO connects suppliers, dropshippers, and wholesale buyers — all in one flow.

---

## The Players

| Role | Who are they? |
|------|--------------|
| **Supplier** | A shop owner who has products and wants to sell through other stores |
| **Dropshipper** | A shop owner who wants to sell products without keeping inventory |
| **Wholesale Buyer** | A business that wants to buy products in bulk directly from suppliers |
| **Zatiq Admin** | The Zatiq team that reviews, routes, and settles everything |
| **Customer** | The end buyer who shops on a dropshipper's store |

---

## The Complete Flow

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│   PHASE 1: SUPPLIER ONBOARDING                                             │
│                                                                             │
│   Shop Owner                                    Zatiq Admin                 │
│       │                                              │                      │
│       │  1. Applies to become a supplier             │                      │
│       │     (business name, details)                 │                      │
│       │─────────────────────────────────────────────>│                      │
│       │                                              │                      │
│       │  2. Uploads verification documents           │                      │
│       │     (trade license, national ID)             │                      │
│       │─────────────────────────────────────────────>│                      │
│       │                                              │                      │
│       │                                 3. Reviews   │                      │
│       │                                    docs      │                      │
│       │                                              │                      │
│       │           Approved / Rejected                │                      │
│       │<─────────────────────────────────────────────│                      │
│       │                                              │                      │
│       ▼                                                                     │
│   Status: Pending ──> Under Review ──> Verified ✓                          │
│                                    └──> Rejected ✗ (can re-apply)          │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│   PHASE 2: SUPPLIER LISTS PRODUCTS                                         │
│                                                                             │
│   Verified Supplier                                                         │
│       │                                                                     │
│       │  4. Picks a product from their own store                           │
│       │                                                                     │
│       │  5. Sets pricing & availability:                                    │
│       │     ┌──────────────────────────────────────────────┐                │
│       │     │  Cost price:           500 BDT (their price) │                │
│       │     │  Suggested retail:     750 BDT (optional)    │                │
│       │     │  Fulfillment time:     48 hours              │                │
│       │     │  Available for dropship?   ✓ Yes             │                │
│       │     │  Available for wholesale?  ✓ Yes             │                │
│       │     │  Wholesale min quantity:   50 units          │                │
│       │     └──────────────────────────────────────────────┘                │
│       │                                                                     │
│       │  6. Product goes live in the ZatiqGO catalog                       │
│       ▼                                                                     │
│                                                                             │
│   ┌────────────────────────────────────────────────────────┐                │
│   │              ZATIQGO PRODUCT CATALOG                   │                │
│   │                                                        │                │
│   │  ★ Boosted Product A  (cost: 300 BDT)  [Premium]     │                │
│   │    Product B           (cost: 500 BDT)                │                │
│   │    Product C           (cost: 200 BDT)                │                │
│   │    Product D           (cost: 800 BDT)  [Wholesale]   │                │
│   │    ...                                                 │                │
│   │                                                        │                │
│   │  Filters: category, price range, delivery speed       │                │
│   └────────────────────────────────────────────────────────┘                │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
                          │                           │
                          ▼                           ▼
          ┌───── DROPSHIP PATH ─────┐   ┌──── WHOLESALE PATH ────┐
          │                         │   │                         │
          ▼                         │   ▼                         │
┌──────────────────────────────┐    │  ┌──────────────────────────────────────┐
│                              │    │  │                                      │
│  PHASE 3A: DROPSHIPPER      │    │  │  PHASE 3B: WHOLESALE ORDER           │
│  ADDS PRODUCT TO STORE      │    │  │                                      │
│                              │    │  │  Wholesale Buyer                     │
│  Dropshipper                 │    │  │      │                               │
│      │                       │    │  │      │ 1. Browses wholesale catalog  │
│      │ 7. Browses catalog    │    │  │      │                               │
│      │    finds a product    │    │  │      │ 2. Places bulk order          │
│      │    (cost: 500 BDT)    │    │  │      │    100 units x 400 BDT       │
│      │                       │    │  │      │    = 40,000 BDT total         │
│      │ 8. "Add to my store"  │    │  │      │                               │
│      │    sets price: 750 BDT│    │  │      │ 3. Pays upfront               │
│      │                       │    │  │      │                               │
│      │ 9. System copies the  │    │  │      ▼                               │
│      │    product into the   │    │  │  Zatiq Admin routes to Supplier      │
│      │    dropshipper's shop │    │  │      │                               │
│      │                       │    │  │      ▼                               │
│      │ 10. Product is live   │    │  │  Supplier ships bulk order           │
│      │     on their store    │    │  │      │                               │
│      │     at 750 BDT        │    │  │      ▼                               │
│      ▼                       │    │  │  Buyer receives delivery             │
│                              │    │  │      │                               │
│  Dropshipper's Store:        │    │  │      ▼                               │
│  ┌────────────────────────┐  │    │  │  Zatiq settles with supplier         │
│  │  Product B    750 BDT  │  │    │  │  (total - commission = supplier pay) │
│  │  [Add to Cart]         │  │    │  │                                      │
│  └────────────────────────┘  │    │  └──────────────────────────────────────┘
│                              │    │
│  Customer sees a normal      │    │
│  product. No idea it's       │    │
│  dropshipped.                │    │
│                              │    │
└──────────────────────────────┘    │
                │                   │
                ▼                   │
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│   PHASE 4: CUSTOMER PLACES AN ORDER                                        │
│                                                                             │
│   Customer                     Dropshipper's Store                          │
│       │                              │                                      │
│       │  11. Browses the store       │                                      │
│       │      adds product to cart    │                                      │
│       │      places order            │                                      │
│       │─────────────────────────────>│                                      │
│       │                              │                                      │
│       │                     12. Order appears in                            │
│       │                         dropshipper's dashboard                     │
│       │                         tagged: "ZatiqGO - Forward Required"        │
│       │                              │                                      │
│       │                              ▼                                      │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│   PHASE 5: ORDER FORWARDING                                                │
│                                                                             │
│   Two modes available:                                                      │
│                                                                             │
│   ┌─────────────────────────┐    ┌──────────────────────────────┐          │
│   │  MANUAL MODE (default)  │    │  AUTO MODE                   │          │
│   │                         │    │                              │          │
│   │  Dropshipper reviews    │    │  Order is automatically     │          │
│   │  the order, then        │    │  forwarded the moment       │          │
│   │  clicks "Forward to     │    │  the customer places it.    │          │
│   │  ZatiqGO"               │    │                              │          │
│   │                         │    │  No action needed from      │          │
│   │  Best for: reviewing    │    │  the dropshipper.           │          │
│   │  each order first       │    │                              │          │
│   └─────────────────────────┘    └──────────────────────────────┘          │
│                                                                             │
│   13. Order is forwarded to ZatiqGO                                        │
│       Status: Pending ──> Forwarded                                        │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│   PHASE 6: ADMIN REVIEW & ROUTING                                          │
│                                                                             │
│   Zatiq Admin                                                               │
│       │                                                                     │
│       │  14. Reviews forwarded order                                       │
│       │                                                                     │
│       │      ┌─────────────────────────────────────────────┐               │
│       │      │  REVIEW GATE — some orders are held:        │               │
│       │      │                                             │               │
│       │      │  • New dropshipper (first few orders)       │               │
│       │      │  • High-value order                         │               │
│       │      │  • Flagged account                          │               │
│       │      │                                             │               │
│       │      │  Trusted dropshipper + normal order?        │               │
│       │      │  → Auto-routed, no hold needed              │               │
│       │      └─────────────────────────────────────────────┘               │
│       │                                                                     │
│       │  15. Admin decides:                                                │
│       │                                                                     │
│       ├──── Route to Supplier ──> Status: Routed                           │
│       ├──── Hold (needs investigation) ──> Status: Under Review            │
│       └──── Reject (fraud/issue) ──> Status: Rejected                      │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼ (if routed)
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│   PHASE 7: SUPPLIER FULFILLMENT                                            │
│                                                                             │
│   Supplier                                                                  │
│       │                                                                     │
│       │  16. Sees new order in their ZatiqGO dashboard                     │
│       │      (product, quantity, shipping address)                          │
│       │                                                                     │
│       │  17. Confirms they can fulfill the order                           │
│       │      Status: Routed ──> Confirmed                                  │
│       │                                                                     │
│       │  18. Packs the product                                             │
│       │      Ships via courier                                             │
│       │      Adds tracking number                                          │
│       │      Status: Confirmed ──> Dispatched                              │
│       │                                                                     │
│       ▼                                                                     │
│   Courier picks up and delivers to customer                                │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│   PHASE 8: DELIVERY & CASH COLLECTION                                      │
│                                                                             │
│   Customer                          Courier                                │
│       │                                │                                    │
│       │  19. Receives the product      │                                    │
│       │      Pays cash on delivery     │                                    │
│       │      (750 BDT)                 │                                    │
│       │───────────────────────────────>│                                    │
│       │                                │                                    │
│       │                       20. Courier sends                             │
│       │                           cash to Zatiq                             │
│       │                                │                                    │
│                                        ▼                                    │
│                                                                             │
│   Zatiq Admin                                                               │
│       │                                                                     │
│       │  21. Marks order as delivered                                      │
│       │      Confirms cash received                                        │
│       │      Status: Dispatched ──> Delivered                              │
│       │                                                                     │
│       ▼                                                                     │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│   PHASE 9: SETTLEMENT (MONEY SPLIT)                                        │
│                                                                             │
│   The system calculates who gets what:                                      │
│                                                                             │
│   ┌─────────────────────────────────────────────────────────────────┐       │
│   │                                                                 │       │
│   │   Customer paid:              750.00 BDT                       │       │
│   │                                                                 │       │
│   │   ┌─────────────────────────────────────────────────────┐       │       │
│   │   │                                                     │       │       │
│   │   │   Supplier gets:          500.00 BDT  (cost price)  │       │       │
│   │   │                                                     │       │       │
│   │   │   Zatiq keeps:             37.50 BDT  (5% comm.)   │       │       │
│   │   │                                                     │       │       │
│   │   │   Dropshipper gets:       212.50 BDT  (the rest)   │       │       │
│   │   │                                                     │       │       │
│   │   └─────────────────────────────────────────────────────┘       │       │
│   │                                                                 │       │
│   │   Formula:                                                      │       │
│   │   Supplier    = cost price x quantity                           │       │
│   │   Commission  = selling price x commission rate                 │       │
│   │   Dropshipper = selling price - supplier cost - commission      │       │
│   │                                                                 │       │
│   └─────────────────────────────────────────────────────────────────┘       │
│                                                                             │
│   22. Zatiq Admin processes disbursements:                                 │
│                                                                             │
│       ├──> Sends 500.00 BDT to Supplier      ✓ Done                       │
│       ├──> Sends 212.50 BDT to Dropshipper   ✓ Done                       │
│       └──> Keeps  37.50 BDT (platform fee)                                │
│                                                                             │
│   Status: Delivered ──> Disbursed ✓                                        │
│                                                                             │
│   ORDER COMPLETE                                                            │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Boost (Paid Visibility)

Suppliers can pay to make their products appear higher in the catalog:

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│   BOOST TIERS                                                   │
│                                                                 │
│   ┌──────────┬───────────────────────────────────────────┐      │
│   │  Basic   │  Higher ranking in search results         │      │
│   ├──────────┼───────────────────────────────────────────┤      │
│   │ Standard │  Featured placement + higher ranking      │      │
│   ├──────────┼───────────────────────────────────────────┤      │
│   │ Premium  │  Top of catalog + featured + badge        │      │
│   └──────────┴───────────────────────────────────────────┘      │
│                                                                 │
│   Admin creates boost campaigns for suppliers.                  │
│   Each campaign has a start date, end date, and tier.           │
│   System tracks impressions (views) and clicks.                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Supplier Tiers

| Tier | Commission | Who qualifies |
|------|-----------|---------------|
| **Standard** | 5% | All new suppliers |
| **Premium** | Negotiated | High volume, consistent quality |
| **Enterprise** | Custom | Top suppliers with dedicated support |

---

## Complete Status Lifecycle

```
DROPSHIP ORDER:

  Pending ──> Forwarded ──> Routed ──> Confirmed ──> Dispatched ──> Delivered ──> Disbursed ✓
                  │
                  └──> Under Review ──> Routed (released)
                                    └──> Rejected / Cancelled ✗


WHOLESALE ORDER:

  Pending ──> Paid ──> Confirmed ──> Dispatched ──> Delivered ──> Settled ✓
                                                               └──> Returned / Failed ✗


SUPPLIER APPLICATION:

  Pending ──> Under Review ──> Verified ✓ ──> (can be Suspended later)
                           └──> Rejected ✗ (can re-apply)


SETTLEMENT:

  Pending ──> Processing ──> Completed ✓
                          └──> Failed (retry)
```
