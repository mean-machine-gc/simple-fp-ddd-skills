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

---

## Navigation structure

The `/docs` directory mirrors the domain:

```
docs/
  _config.yml                          <- Jekyll Just the Docs config (created by ddd-init)
  index.md                             <- Domain home (nav_order: 1)
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

## Pipeline

| # | Name | Type | Description | Failure Codes |
|---|---|---|---|---|
| 1 | `parseCartId` | `STEP` | Validate and parse the raw cart id | `not_a_string`, `empty`, `not_a_uuid` |
| 2 | `parseProductId` | `STEP` | Validate the product identifier | `not_a_string`, `not_a_uuid` |
| 3 | `parseQuantity` | `STEP` | Validate the quantity value | `not_a_number`, `not_positive` |
| 4 | `findCart` | `DEP` | Fetch cart from persistence | -- |
| 5 | `core` | `STEP` | Core domain logic | `cart_empty`, `cart_confirmed`, ... |
| 6 | `saveCart` | `DEP` | Persist the updated cart | -- |

> **STEP** — pure, synchronous domain function. No I/O, fully testable in isolation.
> **DEP** — async infrastructure dependency (persistence or external service).

---

## Decision Table

| Scenario | `parseCartId` :not_a_string | `checkActive` :cart_empty | ... | Outcome |
|---|:---:|:---:|:---:|---|
| OK quantity-reduced | pass | pass | pass | `ActiveCart` — qty reduced |
| FAIL not_a_string | FAIL | -- | -- | Fails: `not_a_string` |
| FAIL cart_empty | pass | FAIL | -- | Fails: `cart_empty` |

> Decision tables show which conditions must pass or fail to produce each outcome.
> A dash (--) means the condition is not evaluated — the pipeline already terminated.
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

### Pipeline — from `steps` array

Direct translation of the `steps` array. Read failure codes from step specs
(`step.spec.shouldFailWith` keys). Steps without specs get `--`.

### Decision Table — from flattened failures + successes

Each success type gets a passing row (all pass). Each failure gets a row showing
where the pipeline short-circuits.

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
create the aggregate `index.md` first:

```md
---
title: Cart
nav_order: 2
has_children: true
---

# Cart

The Cart aggregate manages shopping cart lifecycle — from empty through active
(with items) to confirmed or cancelled.

## Operations

| Operation | Description |
|---|---|
| [Subtract Quantity](subtract-quantity) | Reduce item quantity, remove item, or empty cart |
| [Add Item](add-item) | Add a product to the cart |
| [Remove Item](remove-item) | Remove a product from the cart entirely |
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
- **Every operation page has all sections.** Overview, Interface, Scenarios, Pipeline,
  Decision Table. Atomic functions without steps skip Pipeline.
- **One operation at a time.** Complete the full page before moving on.
- **Front matter is mandatory.** Every `.md` in `/docs` must have Jekyll front matter
  with `title`, `parent` (for operation pages), and `nav_order`.
- **Update the aggregate index** when adding a new operation page.
- **Docs live in `/docs` only.** Never write prose into `.spec.md` files next to code —
  those are fully generated by the CLI.

## Additional resources

- For project conventions, folder structure, and naming rules, see [../ddd-init/reference.md](../ddd-init/reference.md)
