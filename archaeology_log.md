## Vendure Promotion Archaeology Log (Track A — Promotion Challenge)

This log captures concrete **UI Expectation → API Reality** mappings based on Vendure’s documented Admin GraphQL semantics for the Black Friday coupon scenario.

---

### 1. Money Representation

- **UI Expectation**:  
  “Order total above **$100**”.

- **API Reality**:  
  - The `minimumOrderAmount` promotion condition (built‑in in Vendure) expects the threshold as an integer amount in the smallest currency unit (e.g. cents).  
  - In the Admin API this is supplied via a `ConfigArgInput` where:
    - `name = "amount"`
    - `value = "10000"`  
  - The `value` is a **string**, but semantically represents the integer `10000` (100 dollars × 100 cents).

---

### 2. Customer Group vs CustomerGroupId

- **UI Expectation**:  
  “Customer is in **VIP** group” (chosen by visible name in a dropdown).

- **API Reality**:  
  - Promotions operate on **`CustomerGroup` IDs**, not names.
  - The relevant built‑in condition is `inCustomerGroup`, whose argument:
    - `name = "customerGroupId"`
    - `value` must be set to the `id` of the `CustomerGroup` whose `name = "VIP"`.
  - The Admin UI therefore must:
    - Query `customerGroups` from the Admin API.
    - Find the group where `name == "VIP"`.
    - Inject the returned `id` into the promotion condition’s argument.

---

### 3. Conditions as Coded Operations

- **UI Expectation**:  
  Readable condition editors such as:
  - “Order total at least …”
  - “Customer is in group …”

- **API Reality**:  
  - Conditions are expressed as **configurable operations**:
    - Discovered via the `promotionConditions` query.
    - Each condition has a stable `code` (e.g. `minimumOrderAmount`, `inCustomerGroup`) and a list of argument descriptors.
  - When creating a promotion, the Admin API expects:
    - `conditions: [ { code: "...", arguments: [ { name, value }, ... ] }, ... ]`
  - The human‑readable labels in the UI are derived from these codes and their metadata, but the automation layer must work with the **codes and arguments**, not the labels.

---

### 4. Actions as Coded Operations

- **UI Expectation**:  
  An action editor such as “Apply **15%** discount to order”.

- **API Reality**:  
  - Actions are also configurable operations, discovered via `promotionActions`.
  - For this scenario, we rely on the documented built‑in action:
    - `code = "orderPercentageDiscount"`
    - Takes a `discount` argument:
      - `name = "discount"`
      - `value = "15"` (string containing the integer percentage).
  - The Admin UI maps “15%” in the form to this string value in the mutation variables.

---

### 5. Promotions and Enabled State

- **UI Expectation**:  
  A simple toggle like “Enabled” on the promotion form.

- **API Reality**:  
  - The `Promotion` type and `CreatePromotionInput` include a boolean field:
    - `enabled: Boolean`
  - Whether the promotion is active is controlled by this flag, not by creation alone.
  - For the Black Friday coupon to be usable immediately, the Admin UI sets:
    - `enabled = true` in the `createPromotion` mutation input.

---

### 6. Multiple Conditions Composition

- **UI Expectation**:  
  A promotion that should apply only when:
  - Order total is above $100  
  **AND**  
  - Customer is in the `VIP` group.

- **API Reality**:  
  - Vendure’s promotion engine (as documented for built‑in promotions) combines multiple listed conditions with logical **AND** semantics.  
  - In the `createPromotion` input, both conditions are placed in the same `conditions` array:
    - One `minimumOrderAmount` operation.
    - One `inCustomerGroup` operation.
  - The discount action (`orderPercentageDiscount`) is only executed when **all** configured conditions evaluate to true.

---

### 7. Configurable Operation Arguments Are Strings

- **UI Expectation**:  
  - Numeric fields for money and percentages.
  - Dropdown for customer groups.

- **API Reality**:  
  - The `ConfigArgInput` type used for promotion conditions/actions in the Admin API has:
    - `name: String!`
    - `value: String!`
  - Even though arguments are conceptually integers (amount, discount) or IDs (customerGroupId), they are transmitted as **strings** and interpreted according to metadata attached to the operation definition.

---

### 8. Query vs Mutation Responsibilities

- **UI Expectation**:  
  - A single screen where the admin both discovers options (conditions/actions/groups) and saves the promotion.

- **API Reality**:  
  - **Queries**:
    - `promotionConditions` / `promotionActions` provide the available building blocks.
    - `customerGroups` resolves group names to IDs.
  - **Mutation**:
    - `createPromotion` takes the final, composed configuration:
      - Basic fields (name, couponCode, enabled).
      - Conditions (by code + arguments).
      - Actions (by code + arguments).
  - Automation must reproduce this split: lookups first, then a single mutation using the derived IDs and codes.

---

### 9. Assumptions Specific to This Reconstruction

These points are derived from Vendure’s official schema and built‑in promotion documentation. A real deployment might customize or extend these behaviors (e.g. additional conditions or alternative currencies); in such cases, the same network‑driven process must be re‑run and this log and the workflow map updated to match the observed API reality.


