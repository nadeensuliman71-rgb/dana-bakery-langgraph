# Dana's Bakery — LangGraph Order Processing System

## Role and deliverable
Build a complete, runnable Google Colab notebook named `dana_bakery_langgraph.ipynb` using **LangGraph** and deterministic Python only. The notebook should implement Dana's bakery order-processing workflow as an explicit `StateGraph`, display its graph, and execute comprehensive test orders. Do **not** use an LLM, a model API, or an API key. Use the technical specification below rather than treating Dana's original story as the instructions.

## Input contract
The compiled graph must support exactly this invocation style (additional internal state keys may be added by nodes):

```python
result = graph.invoke({
    "item": "babka",
    "quantity": 2,
    "day": "Thursday",
    "city": "Jerusalem",
})
```

Accept item names case-insensitively; normalize whitespace and case. Use canonical lowercase item names in output. `city` may be an empty string or contain surrounding whitespace; treat an empty/whitespace-only city as pickup. Accept `Jerusalem` case-insensitively. Use the seven English weekday names and normalize case. Reject unrecognized days and malformed input with a friendly message rather than crashing. Require `quantity` to be a positive integer (reject booleans, non-integers, zero, and negative values).

## Menu and stock
Use this fixed menu and inventory, representing the snapshot provided by Dana:

| Canonical item | Unit price (ILS) | Available units |
| --- | ---: | ---: |
| `challah` | 18 | 10 |
| `babka` | 45 | 0 |
| `rugelach` (one box) | 30 | 3 |
| `sourdough` (one loaf) | 25 | 4 |

`croissant` is discontinued and must be rejected as an invalid item, even though it appears on an outdated menu. Other unlisted items (e.g. `pizza`) are invalid. Inventory is **not** decremented between independent test invocations; no database or persistence is required.

## Stock checks and substitutions
An item can be sold only if its available stock is **at least** the requested quantity. Never partially fulfill an order.

If insufficient, consider the next product in this fixed sequence:
`babka -> rugelach -> sourdough -> challah`.

A maximum of **two alternative products** may be checked per order (the originally ordered item does not count as an alternative). Keep `quantity` unchanged. If an alternative is accepted, use its own unit price for pricing, return it as `item`, and tell the customer in the final message what changed. If the sequence has no next product, or the second alternative also has insufficient stock, reject the order with a polite explanation. A `challah` shortage therefore rejects immediately, since no next alternative exists. Keep the original product in internal state for message construction. Implement substitution as an explicit graph loop from the substitution node back to stock checking, with a safely bounded number of checks; do not recurse indefinitely.

## Discount and pricing (Avi's corrections take priority)
Dana's earlier statement (10% for totals above 200 ILS) is superseded by Avi's final policy:
- Compute `subtotal = quantity * price_of_actual_item` (baked goods only).
- If `subtotal > 150` **and** the order day is not Friday, apply a 15% discount to `subtotal`.
- At exactly 150 ILS there is **no** discount.
- On Fridays there is **never** a discount.
- Delivery charges are never discounted.
- `total = subtotal - discount + delivery_fee`, using straightforward numeric amounts in ILS. Keep monetary results stable and readable (round to two decimal places if necessary).

## Delivery policy
- An empty/whitespace-only `city`: `delivery = "pickup"` and `delivery_fee = 0`.
- `city = "Jerusalem"` (case-insensitive): `delivery = "delivery"` and `delivery_fee = 20`.
- Any other non-empty city: **do not reject**; change to `delivery = "pickup"` and `delivery_fee = 0`, and clearly mention in the message that delivery outside Jerusalem is unavailable and pickup is required.

## Final output contract
For **every** invocation, including rejected orders, the returned state must contain these six keys:

1. `status`: exactly `"accepted"` or `"rejected"`.
2. `item`: the canonical product actually sold for accepted orders; for rejected orders, retain the last attempted product as a documented convention (no item was sold).
3. `delivery`: exactly `"pickup"` or `"delivery"`; for a rejected order use `"pickup"` by convention because no delivery occurs.
4. `total`: final amount including delivery for accepted orders; `0.0` for rejected orders.
5. `message`: friendly customer-facing text explaining success/substitution/pickup conversion or reason for rejection as relevant. Use **deterministic Python templates**, no LLM calls.
6. `steps`: a chronological list of node names executed during this invocation, including repeated nodes from loops.

These conventions for rejected results and malformed input are implementation decisions to fill gaps in the story, not extra business rules from Dana.

## LangGraph structure and State
Use `TypedDict` (preferably `total=False` for generated/internal fields) to define `State`. Required input and result fields are listed above. You may use internal fields such as `original_item`, `attempts`, `valid`, `in_stock`, `reason`, `subtotal`, `discount`, and `delivery_fee`. Add any additional internal flags needed for clear branching.

Define `steps` with the reducer from the lecture:

```python
from typing import TypedDict, Annotated
import operator

steps: Annotated[list[str], operator.add]
```

Each node must receive the state and return **only the changed fields** as a dictionary, and each visited node must append its own name using `{"steps": ["node_name"]}`. Do not mutate `steps` in place. Avoid resetting the accumulated step list in individual nodes.

Use these eight explicit nodes (or make a compelling, documented equivalent if strictly necessary):
1. `validate_order` — normalize and validate inputs; save `original_item`, initialize substitution counter; mark invalid orders and reason.
2. `check_stock` — check available inventory for the current `item` and full `quantity`.
3. `suggest_alternative` — advance to the next item and increment number of alternatives tried.
4. `calculate_discount` — compute subtotal and discount for the selected product.
5. `handle_delivery` — determine delivery/pickup and final total.
6. `accept_order` — set `status = "accepted"`.
7. `reject_order` — set `status = "rejected"`, `total = 0.0`, `delivery = "pickup"`, and preserve a reason for the customer.
8. `create_message` — produce friendly, informative final `message` for either status.

Use `from langgraph.graph import StateGraph, START, END`. Register every node via `add_node`. Include the following connections:
- `START -> validate_order`
- Conditional route after `validate_order`: valid -> `check_stock`, invalid -> `reject_order`.
- Conditional route after `check_stock`: enough stock -> `calculate_discount`; insufficient stock and eligible alternative remaining -> `suggest_alternative`; otherwise -> `reject_order`.
- `suggest_alternative -> check_stock` (loop).
- `calculate_discount -> handle_delivery -> accept_order -> create_message -> END`.
- `reject_order -> create_message`.

Make routing functions explicit and easy to read. Use `add_conditional_edges`, optionally with `Literal` annotations or a route mapping, so graph branches are understandable when rendered.

## Notebook organization
Include clearly labeled Markdown sections for:
1. Purpose and rules, including the final superseding discount policy.
2. Installation of required libraries (`langgraph` etc.) in a fresh Colab runtime.
3. Imports, menu and stock constants.
4. State schema, nodes, routing functions, graph construction and `compile()`.
5. A graphical diagram **in the notebook** using `graph.get_graph().draw_mermaid_png()` and `IPython.display.Image, display` (display the PNG inline).
6. Run the exact sample invocation specified above and print its result.
7. Test cases with expected outcomes and readable assertions/checks.
8. A brief note explaining how `steps` works and what implementation choices were made where Dana's story was silent.

The notebook must run top-to-bottom from a **fresh Google Colab session** without any secret, API key, external dataset, persistent storage, or LLM.

## Required tests (at least six; include all of these)
Use **independent** invocations. Display the six required result fields for each, and assert expected status/item/delivery/total and the presence/order of `steps`. Include the following test orders:

| Case | Input | Expected result |
| --- | --- | --- |
| 1: original product in stock, free pickup | challah, 2, Monday, empty city | accepted; challah; pickup; 36 ILS |
| 2: first substitute, Jerusalem delivery | babka, 2, Thursday, Jerusalem | accepted; rugelach; delivery; 80 ILS |
| 3: second substitute | babka, 4, Tuesday, empty city | accepted; sourdough; pickup; 100 ILS (rugelach stock 3 is insufficient) |
| 4: two alternatives exhausted | babka, 5, Tuesday, Jerusalem | rejected; total 0 (rugelach and sourdough insufficient) |
| 5: invalid order | pizza, 1, Monday, Jerusalem | rejected; total 0 |
| 6: bad quantity | challah, -2, Monday, empty city | rejected; total 0 |
| 7: non-Jerusalem city | challah, 2, Wednesday, Haifa | accepted; challah; pickup; 36 ILS; explanatory message |
| 8: Avi discount applies | challah, 10, Thursday, Jerusalem | accepted; challah; delivery; 173 ILS (180 - 27 + 20) |
| 9: Friday exception | challah, 10, Friday, Jerusalem | accepted; challah; delivery; 200 ILS |
| 10: no next substitute | challah, 11, Tuesday, empty city | rejected; total 0 |
| 11: discontinued item | croissant, 1, Monday, Jerusalem | rejected; total 0 |
| 12: missing day | challah, 2, "Notaday", empty city | rejected; total 0 |

Ensure that the tests cover **every branch and relevant loop behavior** in the graph. Assertions should fail visibly if the logic violates the story. Also inspect `steps` for specific cases: case 2 has one suggestion; case 3 two suggestions; case 4 rejection after two suggestions; case 5 rejects before stock check.

## Quality constraints
- Prefer simple, readable Python in the style taught in the introductory LangGraph lecture.
- No LLM or API usage; avoid optional external integrations including Instagram.
- Be explicit about the distinction between story requirements and implementation conventions.
- The final output should be the `.ipynb` notebook file, not only a prose description or a `.py` script.
- After creating the notebook, execute it in Colab if access is available, review errors and test outcomes, and fix the notebook until the specified behavior is met. If execution is unavailable, do not claim that it was tested; clearly identify the verification still required.
