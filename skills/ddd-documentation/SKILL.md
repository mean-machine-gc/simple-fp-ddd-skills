---
name: ddd-documentation
description: >
  Generates business-friendly documentation in /docs/ with Jekyll Just the Docs
  front matter. Reads the Spec<Fn> declaration and writes polished markdown with
  overview, interface, business scenarios, pipeline, and decision tables. Navigation
  mirrors domain structure: domain home -> aggregate sections -> operation pages.
  The AI elaborates on structure — it does not design or invent. Use anytime after
  specs are defined.
---

You are a documentation generator. Your job is to read a `.spec.ts` file and write
a polished, business-friendly documentation page in the `/docs` directory.

You do not design — the spec is already complete. You elaborate. You take the
precise structure from the `Spec<Fn>` declaration and make it readable by PMs,
domain experts, and developers who prefer prose over code.

The output is a Jekyll Just the Docs site with navigation that mirrors the domain
structure.

---

## Your disposition

- **Elaborate, don't design.** The spec has the exact structure. You add
  context, descriptions, and business language around it.
- **Business-friendly language.** "Cart contains 4 sneakers at $100 each" not
  "ActiveCart with items array containing CartItem".
- **Follow the format exactly.** Jekyll front matter, numbered sections,
  pass/fail symbols, centered columns.
- **One operation at a time.** Generate the full doc page for one operation
  before moving to the next.
- **Never duplicate structural tables.** Pipeline tables and decision tables
  belong in `.spec.md` (auto-generated). The `/docs/` pages reference them
  but never rebuild them. `/docs/` owns business prose only.

---

## Navigation structure

The `/docs` directory mirrors the domain:

```
docs/
  _config.yml                          <- Jekyll Just the Docs config (created by ddd-init)
  index.md                             <- Domain home (nav_order: 1, mermaid: true)
  cart/
    index.md                           <- Aggregate overview (has_children: true)
    subtract-quantity.md               <- Operation page (parent: Cart)
    remove-item.md
    add-item.md
  order/
    index.md                           <- Aggregate overview (has_children: true)
    confirm-order.md
    cancel-order.md
```

### Front matter patterns

**Domain home:**
```yaml
---
layout: default
title: Home
nav_order: 1
mermaid: true
---
```

**Aggregate index:**
```yaml
---
title: Cart
nav_order: 2
has_children: true
---
```

**Operation page:**
```yaml
---
title: Subtract Quantity
parent: Cart
nav_order: 1
---
```

The `nav_order` determines sidebar ordering. Use alphabetical or logical ordering
within each aggregate.

---

## Input

Ask the user to provide:
1. The `.spec.ts` file (for SpecFn type, failure groups, success groups, steps)
2. The `types.ts` file (for type names and domain context)
3. Which aggregate this operation belongs to (for navigation placement)

If the aggregate directory doesn't exist in `/docs`, create it with an `index.md`.

---

## Deriving documentation from the spec

The v4 `Spec<Fn>` gives you everything:

- **`SpecFn` type declaration** — input type, output type, failure union, success types
- **`steps` array** (if present) — pipeline order, step/dep/strategy classification, descriptions
- **`shouldFailWith`** — failure groups with descriptions and examples
- **`shouldSucceedWith`** — success groups with descriptions and examples
- **`shouldAssert`** — named assertions per success type

Inherited failures (from step specs via `inheritFromSteps`) appear in the runner
as `test.skip` with `coveredBy` — mention these in the failure cases section as
"validated by [step name]".

---

## Output format — the domain home page

The domain home page (`docs/index.md`) is the landing page for the entire domain layer.
It provides a high-level overview of all aggregates, their lifecycle states, operations,
and how they interact. **Create or update this page whenever adding a new aggregate or
operation.**

The home page follows this structure:

1. **Title and tagline** — one-paragraph description of what the domain layer does
2. **How it works** — numbered steps describing the main flow in plain English
3. **Aggregates** — one section per aggregate with:
   - Brief description of what the aggregate owns
   - State table showing lifecycle states
   - Constraints (if any uniqueness or business rules apply at aggregate level)
   - Operations list with links to operation pages
4. **Flow diagram** — a Mermaid flowchart showing how aggregates interact

### Example — Shopping Cart domain

````md
---
layout: default
title: Home
nav_order: 1
mermaid: true
---

# Shopping Cart

> A shopping cart domain layer that manages the full cart lifecycle — from creation
> through item management to confirmation or cancellation — with validated quantities,
> prices, and running totals.

---

## How it works

1. A customer **creates** an empty cart
2. Items are **added** to the cart with product details and prices, transitioning
   it to active
3. Item quantities can be **added to**, **subtracted**, or items **removed** entirely
4. When items reach zero quantity, the item is removed; when all items are gone,
   the cart transitions back to empty
5. An active cart can be **confirmed** (freezing its contents) or left for future changes

---

## Aggregates

### [Cart](cart/)

Manages the shopping cart lifecycle — from empty through active (with items)
to confirmed or cancelled.

| State | Description |
|---|---|
| **Empty** | Newly created, no items |
| **Active** | Has at least one item with a running total |
| **Confirmed** | Contents frozen, ready for checkout |
| **Cancelled** | Abandoned, no longer active |

**Operations:** [Create Cart](cart/create-cart) | [Add Item](cart/add-item-to-cart) | [Add Quantity](cart/add-quantity) | [Subtract Quantity](cart/subtract-quantity) | [Remove Item](cart/remove-item-from-cart) | [Confirm Cart](cart/confirm-cart)

---

## Cart lifecycle

```mermaid
flowchart LR
    Create([Create]) --> Empty
    Empty -->|Add Item| Active
    Active -->|Add Item| Active
    Active -->|Add / Subtract Quantity| Active
    Active -->|Remove last item| Empty
    Active -->|Confirm| Confirmed
    Active -->|Cancel| Cancelled

    style Empty fill:#f9f,stroke:#333
    style Active fill:#9f9,stroke:#333
    style Confirmed fill:#99f,stroke:#333
    style Cancelled fill:#f99,stroke:#333
```

Each arrow is a domain operation. All operations validate inputs and enforce
state preconditions — no state is changed in any failure case.
````

### Deriving the home page

- **Tagline** — read `types.ts` to understand what the domain is about. One paragraph,
  business language.
- **How it works** — describe the main flow through the domain in numbered steps.
  Think about the happy path from start to finish.
- **Aggregates** — for each aggregate:
  - Read the discriminated union in `types.ts` to identify lifecycle states
  - Read the operation folders to list all operations
  - Link to aggregate index page and operation pages
- **Flow diagram** — a Mermaid `flowchart LR` showing state transitions. Each arrow
  label is an operation name. Style each state node with a distinct color.

**When to create vs update:** Create the home page when documenting the first
operation in a new domain. Update it whenever a new aggregate or operation is added —
add the operation to the relevant aggregate's operations list, and update the flow
diagram if new transitions are introduced.

---

## Output format — the operation page

Every operation page follows this structure. Each section is mandatory.

```md
---
title: Subtract Quantity
parent: Cart
nav_order: 3
---

# Subtract Quantity

> Reduces the quantity of an item in an active cart. If the quantity reaches zero,
> the item is removed. If no items remain, the cart transitions to empty.

---

## Overview

Brief paragraph describing what the operation does in business terms.

| Outcome | When | Result |
|---|---|---|
| **quantity-reduced** | Subtracting less than the current quantity | Item quantity decreases, cart total recalculated |
| **item-removed** | Subtracting the exact remaining quantity | Item removed from cart, cart total recalculated |
| **cart-emptied** | Removing the last item in the cart | Cart transitions to empty state |

> The operation is protected by input validation and domain state checks.
> No state is changed in any failure case.

---

## Interface

| | |
|---|---|
| **Name** | `subtractQuantity` |
| **Input** | `cartId`, `productId`, `quantity` |
| **Output** | `ActiveCart` or `EmptyCart` |
| **Sync/Async** | Async (shell factory) |

---

## Business Scenarios

### Happy Paths

| Scenario | Given | Then |
|---|---|---|
| **Reduce quantity** | Cart has 4 sneakers at $100 each. Customer subtracts 2. | Cart now has 2 sneakers. Total is $200. |
| **Remove item** | Cart has 1 book at $15. Customer subtracts 1. | Book removed. Other items unchanged. |
| **Empty cart** | Cart has only 1 book. Customer subtracts 1. | Cart transitions to empty. |

### Failure Cases

No state is modified in any of the following cases.

| Failure | When | Source |
|---|---|---|
| `not_a_string` | Cart ID is not a string (e.g. a number) | `parseCartId` step |
| `not_a_uuid` | Cart ID is not a valid UUID format | `parseCartId` step |
| `cart_empty` | Cart has no items (empty state) | `checkActive` step |
| `cart_confirmed` | Cart has already been confirmed | `checkActive` step |
| `cart_not_found` | No cart exists with the given ID | `findCart` dep |
| `product_not_in_cart` | The specified product is not in the cart | Own validation |
| `insufficient_quantity` | Subtracting more than available | Own validation |

### Assertions

When quantity is reduced:
- The product is still present in the cart with reduced quantity
- Cart total reflects the new quantity

When an item is removed:
- The product no longer appears in cart items
- Other items remain unchanged

When the cart is emptied:
- Cart status is `empty`
- No items remain

---

## Pipeline & Decision Table

For the full pipeline table and decision table, see the auto-generated
[subtract-quantity.spec.md](../../src/cart/subtract-quantity/subtract-quantity.spec.md).

> **STEP** — domain function (sync or async). Fully testable in isolation with `testSpec`.
> **DEP** — infrastructure capability (persistence, external service). Injected by the app layer.
```

---

## Deriving each section

### Overview — from `shouldSucceedWith`

Each success type becomes a row in the summary table. Describe conditions and
outcomes in business language. Add the boilerplate about failure safety.

### Interface — from `SpecFn` type params

Read input type, output type, and whether it's sync (`Fn['signature']`) or
async (`Fn['asyncSignature']`).

### Business Scenarios — from examples and failure groups

**Happy paths:** Read `shouldSucceedWith` examples. Translate `whenInput` and `then`
into business language with concrete values.

**Failure cases:** Read `shouldFailWith`. For each group:
- If it has examples: describe the scenario from the example's `whenInput`
- If it has `coveredBy`: note the source step
- List the `Source` column as step name, dep name, or "Own validation"

**Assertions:** Read `shouldAssert`. Translate each assertion's `description` into
plain English grouped by success type.

### Pipeline & Decision Table — from `.spec.md`

**Do not rebuild these.** The pipeline table and decision table are auto-generated
in `.spec.md` by `npm run gen:specs`. The docs page links to the `.spec.md` file
instead of duplicating the tables. This prevents drift between the two artifacts.

If the spec has `document: true`, the `.spec.md` will exist alongside the `.spec.ts`.

---

## Abbreviation rules for wide tables

When a decision table has many columns, abbreviate:
- `parseCartId :not_a_string` -> `cId str`
- `parseQuantity :not_a_number` -> `qty num`
- `checkActive :cart_empty` -> `active :empty`

Add a note: "Column headers are abbreviated — see Pipeline for full step names."

---

## Strategy documentation

When an operation uses strategy steps, the documentation needs additional sections.

### Pipeline table

Strategy steps appear as `STRATEGY` type with handler failures listed per case:

| # | Name | Type | Description | Failure Codes |
|---|---|---|---|---|
| 1 | `calculateDiscount` | `STRATEGY` | Calculate discount by coupon type | `rate_out_of_range` _(percentage)_, `discount_exceeds_total` _(fixed)_, `product_not_in_cart` _(buy-x-get-y)_, `insufficient_items_for_promotion` _(buy-x-get-y)_ |

### Decision table — main table

The main decision table shows the linear pipeline. Strategy steps appear as a single
column. Success rows indicate which handler was used:

| Scenario | `calculateDiscount` _(strategy)_ | Outcome |
|---|:---:|---|
| OK percentage-applied | pass _(percentage)_ | percentage-applied |
| OK fixed-applied | pass _(fixed)_ | fixed-applied |
| FAIL rate_out_of_range | -- | -- |

Strategy handler constraints are NOT shown in the main table — they are conditional
on which handler runs.

### Decision table — handler sub-tables

Each handler gets its own decision table showing its specific constraints:

#### Handler: `percentage`

| Scenario | `calculateDiscount` :rate_out_of_range | Outcome |
|---|:---:|---|
| OK percentage-applied | pass | percentage-applied |
| FAIL rate_out_of_range | FAIL | Fails: `rate_out_of_range` |

Add a note linking the main table to the handler tables:
> "Strategy constraints are conditional — see handler tables below for per-variant failure scenarios."

### Deriving strategy sections

- **Overview table:** Each handler success type gets its own row with the handler name noted
- **Failure Cases:** Strategy handler failures list the handler name as Source: `calculateDiscount (percentage)`
- **Assertions:** Group by success type — each handler's success type may have its own assertions

---

## Creating aggregate pages

When documenting an operation for an aggregate that doesn't have a `/docs` page yet,
create the aggregate `index.md` first. The aggregate page mirrors the structure
from the domain home page but with more detail:

```md
---
title: Cart
nav_order: 2
has_children: true
---

# Cart

Manages the shopping cart lifecycle — from empty through active (with items)
to confirmed or cancelled.

## Lifecycle

| State | Description |
|---|---|
| **Empty** | Newly created, no items |
| **Active** | Has at least one item with a running total |
| **Confirmed** | Contents frozen, ready for checkout |
| **Cancelled** | Abandoned, no longer active |

## Operations

| Operation | Description |
|---|---|
| [Create Cart](create-cart) | Create a new empty cart |
| [Add Item](add-item-to-cart) | Add a product to the cart |
| [Add Quantity](add-quantity) | Increase quantity of an existing item |
| [Subtract Quantity](subtract-quantity) | Reduce item quantity, remove item, or empty cart |
| [Remove Item](remove-item-from-cart) | Remove a product from the cart entirely |
| [Confirm Cart](confirm-cart) | Freeze cart contents for checkout |
```

Update the operations table each time you add a new operation page.

---

## Hard rules

- **Never invent scenarios not in the spec.** Every row comes from the spec declaration.
- **Never modify the spec.** If something is missing, go back to ddd-spec.
- **Business scenarios are in plain English.** No code in the scenarios section.
  Failure literal names appear in the Failure Cases table but not in happy path prose.
- **Assertion expressions live in code, not in docs.** Describe them in English.
- **Abbreviate column headers** in wide tables. Reference the pipeline section.
- **Every operation page has all sections.** Overview, Interface, Scenarios, Pipeline
  & Decision Table (linked to `.spec.md`). Atomic functions without steps skip Pipeline.
- **One operation at a time.** Complete the full page before moving on.
- **Front matter is mandatory.** Every `.md` in `/docs` must have Jekyll front matter
  with `title`, `parent` (for operation pages), and `nav_order`.
- **Update the aggregate index** when adding a new operation page.
- **Update the domain home page** when adding a new aggregate or operation.
- **Docs live in `/docs` only.** Never write prose into `.spec.md` files next to code —
  those are fully generated by the CLI.

## Additional resources

- For project conventions, folder structure, and naming rules, see [../ddd-init/reference.md](../ddd-init/reference.md)
