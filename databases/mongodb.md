# MongoDB

## Table of Contents
- [What is MongoDB?](#what-is-mongodb)
  - [Why/when people reach for MongoDB](#whywhen-people-reach-for-mongodb)
  - [Where it's a weaker fit](#where-its-a-weaker-fit)
  - [Common Use Cases](#common-use-cases)
  - [Key terms to know before going further](#key-terms-to-know-before-going-further)
- [MongoDB Data Modeling: Embed vs Reference](#mongodb-data-modeling-embed-vs-reference)
  - [TL;DR](#tldr)
  - [Context](#context)
  - [The 11 Guidelines](#the-11-guidelines)
    - [1. Simplicity](#1-simplicity)
    - [2. Go Together ("has-a" / "contains")](#2-go-together-has-a--contains)
    - [3. Query Atomicity](#3-query-atomicity)
    - [4. Update Complexity](#4-update-complexity)
    - [5. Archival](#5-archival)
    - [6. Cardinality](#6-cardinality)
    - [7. Data Duplication](#7-data-duplication)
    - [8. Document Size](#8-document-size)
    - [9. Document Growth](#9-document-growth)
    - [10. Workload](#10-workload)
    - [11. Individuality](#11-individuality)
  - [Summary Table](#summary-table)
  - [Worked Example: Applying All 11 Guidelines](#worked-example-applying-all-11-guidelines)
  - [Quick Decision Framework](#quick-decision-framework)
  - [Reference](#reference)

---

## What is MongoDB?
MongoDB is a **document-oriented NoSQL database**. Instead of storing data in rows and tables like a relational (SQL) database, it stores data as **BSON documents** (a binary form of JSON) grouped into **collections**.

- **Document** — a single record, structurally similar to a JSON object (e.g., one user, one order, one product). Can contain nested objects and arrays.
- **Collection** — a group of documents, roughly analogous to a "table" in SQL, but without a fixed schema — documents in the same collection can have different fields.
- **Database** — a group of collections.

```json
// Example document in a "users" collection
{
  "_id": "64f1a2b3c4d5e6f7a8b9c0d1",
  "name": "Kaushik Karan",
  "email": "kaushikkaran@example.com",
  "address": {
    "city": "Bengaluru",
    "pincode": "560001"
  },
  "tags": ["premium", "early-adopter"]
}
```

### Why/when people reach for MongoDB
- **Flexible/evolving schema** — good fit when your data structure isn't fully fixed upfront or varies between records (e.g., product catalogs with wildly different attributes per category).
- **Natural fit for hierarchical data** — nested objects/arrays map closely to how you'd model data in application code (JSON-like), reducing the "impedance mismatch" you get with relational tables.
- **Horizontal scalability** — built-in sharding makes it easier to scale out across servers for very large datasets or high write throughput, compared to traditionally scaling a single relational DB instance.
- **High write/read throughput** for semi-structured data at scale (e.g., logging, content management, catalogs, real-time analytics).

### Where it's a weaker fit
- Heavy **multi-document transactional** consistency needs across many entities (though MongoDB does support multi-document ACID transactions since v4.0, it's not the primary strength).
- Data that is deeply relational/normalized by nature, with lots of many-to-many joins (e.g., complex financial ledgers) — a relational DB's JOINs and constraints often model this more naturally.
- Teams that want the DB engine to strictly enforce schema/data integrity rather than the application layer.

### Common Use Cases
A few concrete scenarios where MongoDB is a natural fit:

- **Content management systems (CMS):** Blog posts, articles, or product pages often have varying structures (some have videos, some have image galleries, some have neither). A flexible schema avoids constant `ALTER TABLE` migrations that a relational DB would need.
- **Product catalogs (e-commerce):** A `Shoe` and a `Laptop` have completely different attributes (size/color vs RAM/CPU). Storing both in the same `products` collection is natural in MongoDB; forcing them into one rigid SQL table would mean dozens of nullable columns.
- **User profiles / personalization:** Each user might have a different set of preferences, settings, or activity metadata. Embedding this as a flexible sub-document is simpler than a rigid schema.
- **Real-time analytics / IoT / event logging:** High-volume writes (sensor data, clickstreams, app events) benefit from MongoDB's write throughput and horizontal scaling via sharding.
- **Mobile/single-page app backends:** Since documents map closely to JSON, MongoDB integrates naturally with app code — you often store data in roughly the same shape you'll consume it in the UI, reducing transformation logic.

> **Example scenario:** An e-commerce startup building a product catalog where categories range from electronics to clothing. Each category has different attributes (electronics need `warranty`, `specs`; clothing needs `size`, `material`). Modeling this in a relational DB would require either a huge sparse table or an over-normalized EAV (entity-attribute-value) design. In MongoDB, each product is just a document with whatever fields make sense for its category — no schema migration needed when a new category is added.

### Key terms to know before going further
| SQL term | MongoDB equivalent |
|---|---|
| Table | Collection |
| Row | Document |
| Column | Field |
| Primary Key | `_id` field |
| JOIN | `$lookup` (aggregation) or manual reference resolution |

The single biggest modeling decision in MongoDB — and the one covered in this doc — is deciding **when to embed related data inside a document vs. when to reference it in a separate collection**. Unlike SQL (where normalization is the default), MongoDB's flexible document structure means this choice is yours to make deliberately, and it has real performance and maintainability consequences.

---

# MongoDB Data Modeling: Embed vs Reference

## TL;DR
When modeling relationships in MongoDB, you have two choices: **embed** (nest the related data inside the same document) or **reference** (store related data in a separate collection and link via an ID). MongoDB provides 11 guidelines to help decide. Run through them for any "has-a" relationship in your schema — if most guidelines point to **Embed**, embed it. If key ones point to **Reference** (especially Update Complexity or Archival), reference it.

## Context
This decision comes up constantly when designing collections — e.g., should a `User` document contain its `addresses` array, or should addresses live in a separate `addresses` collection referenced by `userId`? Getting this wrong leads to either bloated, slow-to-update documents (over-embedding) or excessive joins/lookups that hurt read performance (over-referencing).

---

## The 11 Guidelines

Each guideline is a yes/no question. Below, each includes the rule, a plain-language explanation, and a worked example.

### 1. Simplicity
**Question:** Would keeping the pieces of information together lead to a simpler data model and code?
**Yes → Embed**

If embedding makes your application logic dramatically simpler (fewer collections to manage, fewer joins), lean toward embedding.

> **Example:** A `BlogPost` document embedding its `comments` array is simpler to fetch and render on a post page than querying a separate `comments` collection and joining it in code.

---

### 2. Go Together ("has-a" / "contains")
**Question:** Do the pieces of information have a "has-a," "contains," or similar relationship?
**Yes → Embed**

If one entity logically *contains* the other (not just *relates to* it), embedding reflects that natural structure.

> **Example:** An `Order` document has-a shipping `address` and has-a list of `lineItems`. These aren't independent entities — they're parts of the order. Embed them.

---

### 3. Query Atomicity
**Question:** Does the application query the pieces of information together?
**Yes → Embed**

If your app almost always reads both pieces of data in the same query/screen, embedding avoids a second round trip.

> **Example:** When displaying a `Product` page, you always need its `reviews summary` (average rating, count) together with the product details. Embed the summary in the product document.

---

### 4. Update Complexity
**Question:** Are the pieces of information updated together?
**No → Reference**

If the two pieces of data are updated **independently**, or at very different frequencies, embedding causes unnecessary write amplification (rewriting a large document just to update a small field).

> **Example:** A `Product` document and its `inventory count` — inventory changes constantly (every sale) while product details (name, description) rarely change. Keep `inventory` in a **separate referenced collection** so frequent stock updates don't touch the bulkier product document.

---

### 5. Archival
**Question:** Should the pieces of information be archived at the same time?
**No → Reference**

If one piece of data needs to be archived/deleted independently of the other (different lifecycle/retention rules), separate them.

> **Example:** A `User` account should persist indefinitely, but their `activityLogs` might need to be archived or purged after 90 days per a retention policy. Reference logs in a separate collection so archival doesn't touch the user record.

---

### 6. Cardinality
**Question:** Is there a high cardinality (current or growing) in the child side of the relationship?
**No → Embed** (i.e., embed only when cardinality is low/bounded)

If the "many" side of a one-to-many relationship is small and bounded, embedding is fine. If it's large or unbounded, reference instead.

> **Example:** A `Person` document embedding a handful of `phoneNumbers` (typically 1-3) is fine — low cardinality. But a `Celebrity` document with millions of `followers` should **not** embed followers — that's the classic high-cardinality case requiring a separate `followers` collection referencing the celebrity's ID.

---

### 7. Data Duplication
**Question:** Would data duplication be too complicated to manage and undesired?
**No → Embed** (duplication is fine/manageable → embed)

Embedding often means denormalizing (duplicating) data across documents. If that duplication is small and rarely changes, it's an acceptable trade-off for read performance.

> **Example:** Embedding an `authorName` inside every `BlogPost` document (in addition to referencing `authorId`) duplicates the author's name across many posts. Since author names rarely change, this duplication is cheap and avoids a join on every post read — embed it.

---

### 8. Document Size
**Question:** Would the combined size of the pieces of information take too much memory or transfer bandwidth for the application?
**No → Embed** (small combined size → embed)

MongoDB has a 16MB per-document limit, and large documents are also costly to transfer and cache. If combining the data keeps documents reasonably small, embed.

> **Example:** Embedding a customer's `shippingAddress` (a few fields) inside an `Order` document adds negligible size — embed it. But embedding the full text of every `Review` (potentially thousands of long reviews) inside a `Product` document could balloon document size — reference those instead.

---

### 9. Document Growth
**Question:** Would the embedded piece grow without bound?
**No → Embed** (bounded growth → embed)

Arrays that grow indefinitely eventually hit the document size limit and cause performance issues (constant document reallocation). If growth is naturally capped, embedding is safe.

> **Example:** Embedding a `top 5 tags` array in a `Product` document is safe — bounded at 5 items. Embedding an ever-growing `orderHistory` array inside a `Customer` document is risky — over years, this could grow unbounded. Reference orders in a separate `orders` collection instead, linked by `customerId`.

---

### 10. Workload
**Question:** Are the pieces of information written at different times in a write-heavy workload?
**No → Embed** (written together / not write-heavy → embed)

If one part of the data is updated far more frequently than the other in a high-throughput write scenario, separating them avoids lock contention and unnecessary rewrites of the larger document.

> **Example:** A `Product` document (rarely updated) versus its real-time `stock level` (updated on every single sale in a high-traffic store). In a write-heavy workload, keep `stock level` in its own referenced document so frequent writes don't repeatedly touch the full product record.

---

### 11. Individuality
**Question:** For the child side of the relationship, can the pieces exist by themselves without a parent?
**No → Embed** (child cannot exist independently → embed)

If the "child" data has no meaning or use outside the context of its parent, it belongs embedded inside it.

> **Example:** A `lineItem` (product + quantity + price) only makes sense in the context of its parent `Order` — it can't exist independently. Embed it. Conversely, a `Product` in a catalog can absolutely exist independently of any single order — so `Product` should be a separate, referenced collection, not embedded inside every order.

---

## Summary Table

| Guideline | Question | Answer that suggests... | Result |
|---|---|---|---|
| Simplicity | Would embedding simplify the model/code? | Yes | **Embed** |
| Go Together | Is it a "has-a"/"contains" relationship? | Yes | **Embed** |
| Query Atomicity | Queried together? | Yes | **Embed** |
| Update Complexity | Updated together? | No | **Reference** |
| Archival | Archived together? | No | **Reference** |
| Cardinality | High/growing cardinality on child side? | No | **Embed** |
| Data Duplication | Would duplication be too complex? | No | **Embed** |
| Document Size | Combined size too large? | No | **Embed** |
| Document Growth | Would embedded piece grow unbounded? | No | **Embed** |
| Workload | Written at different times, write-heavy? | No | **Embed** |
| Individuality | Can child exist without parent? | No | **Embed** |

**Note the pattern:** 9 of the 11 guidelines lean toward embedding by default — MongoDB's philosophy favors embedding unless a specific concern (mainly **update frequency** or **archival lifecycle**) says otherwise. Those two are usually the deciding factors in practice.

---

## Worked Example: Applying All 11 Guidelines

**Scenario:** Modeling `User` and their `Orders` in an e-commerce app.

| Guideline | Analysis | Verdict |
|---|---|---|
| Simplicity | Orders as a separate collection is simpler than one giant user doc | Reference |
| Go Together | A user "has" orders, but it's a loose relationship, not core identity | Reference |
| Query Atomicity | User profile page rarely needs to show *all* past orders inline | Reference |
| Update Complexity | Orders are created/updated independently of user profile edits | **Reference** |
| Archival | Orders may need 7-year retention for compliance; user profile doesn't | **Reference** |
| Cardinality | A user can have hundreds/thousands of orders over time — high cardinality | **Reference** |
| Document Growth | Order history would grow unbounded over a user's lifetime | **Reference** |

**Decision:** Reference — store `orders` in their own collection with a `userId` field, rather than embedding an `orders` array inside `User`. This is a textbook one-to-many-with-high-cardinality case.

Compare this to embedding a `shippingAddress` inside a single `Order` — that one clearly leans **Embed** (has-a relationship, queried together, bounded size, doesn't exist independently of the order).

---

## Quick Decision Framework

1. Start by assuming **embed** (MongoDB's default bias).
2. Check **Update Complexity** and **Archival** first — these are the strongest signals to switch to reference.
3. Check **Cardinality** and **Document Growth** next — unbounded/high-cardinality children should almost always be referenced.
4. If none of those trigger, the remaining guidelines (Simplicity, Go Together, Query Atomicity, Data Duplication, Document Size, Workload, Individuality) will almost always confirm **embed**.

## Reference
MongoDB's official embedding vs. referencing decision guidelines (MongoDB University / MongoDB Manual — Data Modeling).
